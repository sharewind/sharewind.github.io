---
layout: post
title: "jafka troubleshooting"
date: 2013-03-15 10:57
comments: true
categories: jafka zookeeper
---

#### 背景

通过 jafka-watch 查看当前topic 及相应的consumer 情况，发现总有许多topic的某些partition 分片居然没有consumer 去消费。

#### 过程

1. 刚开始认为是不是有的消费者没有连接上，于是把所有消费者都重新启动了一遍

2. 重新启动完了以后，部分topic的所有分片是都有消费者消费，但某些topic还是存在分片没有人消费的情况。

3. 确认所有消费者都在zookeeper的节点上注册了，于是怀疑 jafka 的 客户端的负载均衡算法有问题。


class:ZookeeperConsumerConnector

        private boolean rebalance(Cluster cluster) {

            // map for current consumer: topic->[groupid-consumer-0,groupid-consumer-1,...,groupid-consumer-N]
			// 获取当前进程的消费者
            Map<String, Set<String>> myTopicThreadIdsMap = ZkUtils.getTopicCount(zkClient, group, consumerIdString)
                    .getConsumerThreadIdsPerTopic();

            // map for all consumers in this group: topic->[groupid-consumer1-0,...,groupid-consumerX-N]
			// 获取当前group的所有消费者，会根据名称进行排序
            Map<String, List<String>> consumersPerTopicMap = ZkUtils.getConsumersPerTopic(zkClient, group);
            // map for all broker-partitions for the topics in this consumerid: topic->[brokerid0-partition0,...,brokeridN-partitionN]
			// 获取该topic的所有分片信息
            Map<String, List<String>> brokerPartitionsPerTopicMap = ZkUtils.getPartitionsForTopics(zkClient,
                    myTopicThreadIdsMap.keySet());
            /**
             * fetchers must be stopped to avoid data duplication, since if the current
             * rebalancing attempt fails, the partitions that are released could be owned by
             * another consumer. But if we don't stop the fetchers first, this consumer would
             * continue returning data for released partitions in parallel. So, not stopping
             * the fetchers leads to duplicate data.
             */
            closeFetchers(cluster, messagesStreams, myTopicThreadIdsMap);
            releasePartitionOwnership(topicRegistry);
            //
            Map<StringTuple, String> partitionOwnershipDecision = new HashMap<StringTuple, String>();
            Pool<String, Pool<Partition, PartitionTopicInfo>> currentTopicRegistry = new Pool<String, Pool<Partition, PartitionTopicInfo>>();

			// 分配核心算法
            for (Map.Entry<String, Set<String>> e : myTopicThreadIdsMap.entrySet()) {
                final String topic = e.getKey();
                currentTopicRegistry.put(topic, new Pool<Partition, PartitionTopicInfo>());
                //
                ZkGroupTopicDirs topicDirs = new ZkGroupTopicDirs(group, topic);
                List<String> curConsumers = consumersPerTopicMap.get(topic);
                List<String> curBrokerPartitions = brokerPartitionsPerTopicMap.get(topic);

				// 每个消费者消费的分片数
				// 如果只有一个消费者线程，也可以消费所有分片
                final int nPartsPerConsumer = curBrokerPartitions.size() / curConsumers.size();
                final int nConsumersWithExtraPart = curBrokerPartitions.size() % curConsumers.size();

                logger.info("Consumer " + consumerIdString + " rebalancing the following partitions:\n    " + curBrokerPartitions + "\nfor topic " + topic + " with consumers:\n    " + curConsumers);
                if (logger.isDebugEnabled()) {
                    StringBuilder buf = new StringBuilder(1024);
                    buf.append("[").append(topic).append("] preassigning details:");
                    for (int i = 0; i < curConsumers.size(); i++) {
                        final int startPart = nPartsPerConsumer * i + Math.min(i, nConsumersWithExtraPart);
                        final int nParts = nPartsPerConsumer + ((i + 1 > nConsumersWithExtraPart) ? 0 : 1);
                        if (nParts > 0) {
                            for (int m = startPart; m < startPart + nParts; m++) {
                                buf.append("\n    ").append(curConsumers.get(i)).append(" ==> ")
                                        .append(curBrokerPartitions.get(m));
                            }
                        }
                    }
                    logger.debug(buf.toString());
                }

                //consumerThreadId=> groupid_consumerid-index (index from count)
                for (String consumerThreadId : e.getValue()) {
					// 消费的分片与当前消费者线程id 在 所有消费线程id的index有关
                    final int myConsumerPosition = curConsumers.indexOf(consumerThreadId);
                    assert (myConsumerPosition >= 0);
                    final int startPart = nPartsPerConsumer * myConsumerPosition + Math.min(myConsumerPosition,
                            nConsumersWithExtraPart);
                    final int nParts = nPartsPerConsumer + ((myConsumerPosition + 1 > nConsumersWithExtraPart) ? 0 : 1);

                    /**
                     * Range-partition the sorted partitions to consumers for better locality.
                     * The first few consumers pick up an extra partition, if any.
                     */
                    if (nParts <= 0) {
                        logger.warn("No broker partitions consumed by consumer thread " + consumerThreadId + " for topic " + topic);
                    } else {
                        for (int i = startPart; i < startPart + nParts; i++) {
                            String brokerPartition = curBrokerPartitions.get(i);
                            logger.info("[" + consumerThreadId + "] ==> " + brokerPartition + " claimming");
                            addPartitionTopicInfo(currentTopicRegistry, topicDirs, brokerPartition, topic,
                                    consumerThreadId);
                            // record the partition ownership decision
                            partitionOwnershipDecision.put(new StringTuple(topic, brokerPartition), consumerThreadId);
                        }
                    }
                }
            }
            //
            /**
             * move the partition ownership here, since that can be used to indicate a truly
             * successful rebalancing attempt A rebalancing attempt is completed successfully
             * only after the fetchers have been started correctly
             */
            if (reflectPartitionOwnershipDecision(partitionOwnershipDecision)) {
                logger.debug("Updating the cache");
                logger.debug("Partitions per topic cache " + brokerPartitionsPerTopicMap);
                logger.debug("Consumers per topic cache " + consumersPerTopicMap);
                topicRegistry = currentTopicRegistry;
                updateFetcher(cluster, messagesStreams);
                return true;
            } else {
                return false;
            }
            ////////////////////////////
        }
 
4.配置jafka 的logger 信息为INFO 状态， 发现居然返回的topic 消费者列表比实际存在的数量还多。

5.难道是zookeeper 的配置信息出错，于是遍历所有zookeeper 服务器，获取消费者列表。发现其中一台zookeeper 服务器的数据居然与 其它zookeeper服务器数据不一致，于是坑爹了。 zookeeper 这个号称提供 数据一致性 服务的东东，居然还能出现数据不一致的bug.

