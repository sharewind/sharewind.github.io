---
layout: post
title: "Selector Provider SPI"
date: 2013-08-27 11:51
comments: true
categories: Java
---

Selector Provider SPI

{% codeblock Opens a selector. lang:java %}

public abstract class Selector implements Closeable {

    /**
     * Initializes a new instance of this class.
     */
    protected Selector() { }

    /**
     * Opens a selector.
     *
     * <p> The new selector is created by invoking the {@link
     * java.nio.channels.spi.SelectorProvider#openSelector openSelector} method
     * of the system-wide default {@link
     * java.nio.channels.spi.SelectorProvider} object.  </p>
     *
     * @return  A new selector
     *
     * @throws  IOException
     *          If an I/O error occurs
     */
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

{% endcodeblock %}



{% codeblock SelectorProvider lang:java %}

public abstract class SelectorProvider {

    private static final Object lock = new Object();
    private static SelectorProvider provider = null;

    /**
     * Initializes a new instance of this class.  </p>
     *
     * @throws  SecurityException
     *          If a security manager has been installed and it denies
     *          {@link RuntimePermission}<tt>("selectorProvider")</tt>
     */
    protected SelectorProvider() {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null)
            sm.checkPermission(new RuntimePermission("selectorProvider"));
    }

    private static boolean loadProviderFromProperty() {
        String cn = System.getProperty("java.nio.channels.spi.SelectorProvider");
        if (cn == null)
            return false;
        try {
			// 通过反射加载类，并创建类的实例
            Class<?> c = Class.forName(cn, true,
                                       ClassLoader.getSystemClassLoader());
            provider = (SelectorProvider)c.newInstance();
            return true;
        } catch (ClassNotFoundException x) {
            throw new ServiceConfigurationError(null, x);
        } catch (IllegalAccessException x) {
            throw new ServiceConfigurationError(null, x);
        } catch (InstantiationException x) {
            throw new ServiceConfigurationError(null, x);
        } catch (SecurityException x) {
            throw new ServiceConfigurationError(null, x);
        }
    }

    private static boolean loadProviderAsService() {

		// 这里通过 ServiceLoader 加载 SelectorProvider
        ServiceLoader<SelectorProvider> sl =
            ServiceLoader.load(SelectorProvider.class,
                               ClassLoader.getSystemClassLoader());
        Iterator<SelectorProvider> i = sl.iterator();

		// XXX 这里为什么要使用一个for 循环呢，底下不是只判断一次就可以了？
		// 并且实际上也只执行一次？？？
        for (;;) {
            try {
                if (!i.hasNext())
                    return false;
                provider = i.next();
                return true;
            } catch (ServiceConfigurationError sce) {
                if (sce.getCause() instanceof SecurityException) {
                    // Ignore the security exception, try the next provider
                    continue;
                }
                throw sce;
            }
        }
    }

    /**
     * Returns the system-wide default selector provider for this invocation of
     * the Java virtual machine.
     *
     * <p> The first invocation of this method locates the default provider
     * object as follows: </p>
     *
     * <ol>
     *
     *   <li><p> If the system property
     *   <tt>java.nio.channels.spi.SelectorProvider</tt> is defined then it is
     *   taken to be the fully-qualified name of a concrete provider class.
     *   The class is loaded and instantiated; if this process fails then an
     *   unspecified error is thrown.  </p></li>
     *
     *   <li><p> If a provider class has been installed in a jar file that is
     *   visible to the system class loader, and that jar file contains a
     *   provider-configuration file named
     *   <tt>java.nio.channels.spi.SelectorProvider</tt> in the resource
     *   directory <tt>META-INF/services</tt>, then the first class name
     *   specified in that file is taken.  The class is loaded and
     *   instantiated; if this process fails then an unspecified error is
     *   thrown.  </p></li>
     *
     *   <li><p> Finally, if no provider has been specified by any of the above
     *   means then the system-default provider class is instantiated and the
     *   result is returned.  </p></li>
     *
     * </ol>
     *
     * <p> Subsequent invocations of this method return the provider that was
     * returned by the first invocation.  </p>
     *
     * @return  The system-wide default selector provider
     */
    public static SelectorProvider provider() {
		// 同步锁
        synchronized (lock) {
            if (provider != null)
                return provider;
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
							// 从系统属性配置中加载
                            if (loadProviderFromProperty())
                                return provider;
							// 以SPI的方式加载
                            if (loadProviderAsService())
                                return provider;
							
							// 返回默认的SelectorProvider
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }

{% endcodeblock %}


{% codeblock ServiceLoader lang:java %}

	public final class ServiceLoader<S> implements Iterable<S>{
	
	    private static final String PREFIX = "META-INF/services/";
	
		// 服务接口的class
	    // The class or interface representing the service being loaded
	    private Class<S> service;
	
	    // The class loader used to locate, load, and instantiate providers
	    private ClassLoader loader;
	
		// 根据实例化的顺序缓存providers
	    // Cached providers, in instantiation order
	    private LinkedHashMap<String,S> providers = new LinkedHashMap<String,S>();
	
	    // The current lazy-lookup iterator
	    private LazyIterator lookupIterator;
	
		//  清除此加载器的服务者缓存，以重载所有服务者。
	    /**
	     * Clear this loader's provider cache so that all providers will be
	     * reloaded.
	     *
	     * <p> After invoking this method, subsequent invocations of the {@link
	     * #iterator() iterator} method will lazily look up and instantiate
	     * providers from scratch, just as is done by a newly-created loader.
	     *
	     * <p> This method is intended for use in situations in which new providers
	     * can be installed into a running Java virtual machine.
	     */
	    public void reload() {
	        providers.clear();// 清除缓存
	        lookupIterator = new LazyIterator(service, loader);
	    }
	
		// 私有构造函数
	    private ServiceLoader(Class<S> svc, ClassLoader cl) {
	        service = svc;
	        loader = cl;
	        reload();
	    }
	
	
		// 针对给定服务类型和类加载器创建新的服务加载器。
	    /**
	     * Creates a new service loader for the given service type and class
	     * loader.
	     *
	     * @param  service
	     *         The interface or abstract class representing the service
	     *
	     * @param  loader
	     *         The class loader to be used to load provider-configuration files
	     *         and provider classes, or <tt>null</tt> if the system class
	     *         loader (or, failing that, the bootstrap class loader) is to be
	     *         used
	     *
	     * @return A new service loader
	     */
	    public static <S> ServiceLoader<S> load(Class<S> service,
	                                            ClassLoader loader){
	        return new ServiceLoader<S>(service, loader);
	    }

		
		// 针对给定服务类型创建新的服务加载器，使用当前线程的上下文类加载器。
	    /**
	     * Creates a new service loader for the given service type, using the
	     * current thread's {@linkplain java.lang.Thread#getContextClassLoader
	     * context class loader}.
	     *
	     * <p> An invocation of this convenience method of the form
	     *
	     * <blockquote><pre>
	     * ServiceLoader.load(<i>service</i>)</pre></blockquote>
	     *
	     * is equivalent to
	     *
	     * <blockquote><pre>
	     * ServiceLoader.load(<i>service</i>,
	     *                    Thread.currentThread().getContextClassLoader())</pre></blockquote>
	     *
	     * @param  service
	     *         The interface or abstract class representing the service
	     *
	     * @return A new service loader
	     */
	    public static <S> ServiceLoader<S> load(Class<S> service) {
	        ClassLoader cl = Thread.currentThread().getContextClassLoader();
	        return ServiceLoader.load(service, cl);
	    }
	
		// 针对给定服务类型创建新的服务加载器，使用扩展类加载器。
	    /**
	     * Creates a new service loader for the given service type, using the
	     * extension class loader.
	     *
	     * <p> This convenience method simply locates the extension class loader,
	     * call it <tt><i>extClassLoader</i></tt>, and then returns
	     *
	     * <blockquote><pre>
	     * ServiceLoader.load(<i>service</i>, <i>extClassLoader</i>)</pre></blockquote>
	     *
	     * <p> If the extension class loader cannot be found then the system class
	     * loader is used; if there is no system class loader then the bootstrap
	     * class loader is used.
	     *
	     * <p> This method is intended for use when only installed providers are
	     * desired.  The resulting service will only find and load providers that
	     * have been installed into the current Java virtual machine; providers on
	     * the application's class path will be ignored.
	     *
	     * @param  service
	     *         The interface or abstract class representing the service
	     *
	     * @return A new service loader
	     */
	    public static <S> ServiceLoader<S> loadInstalled(Class<S> service) {
	        ClassLoader cl = ClassLoader.getSystemClassLoader();
	        ClassLoader prev = null;
	        while (cl != null) {
	            prev = cl;
	            cl = cl.getParent();
	        }
	        return ServiceLoader.load(service, prev);
	    }

{% endcodeblock %}


{% codeblock ServiceLoader--》LazyIterator lang:java %}

    private class LazyIterator implements Iterator<S>{

        Class<S> service;
        ClassLoader loader;
        Enumeration<URL> configs = null;
        Iterator<String> pending = null;
        String nextName = null;

        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }

        public boolean hasNext() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
					// 配置文件的完整文件名
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
			
			// 解析文件
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
			// 得到nextName
            nextName = pending.next();
            return true;
        }

        public S next() {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            String cn = nextName;
            nextName = null;
            try {
				// 加载相应的服务实现类
                S p = service.cast(Class.forName(cn, true, loader)
                                   .newInstance());
				// 放入缓存中
                providers.put(cn, p);
                return p;
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated: " + x,
                     x);
            }
            throw new Error();          // This cannot happen
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    }

	// 以延迟方式加载此加载器服务的可用提供者。
    /**
     * Lazily loads the available providers of this loader's service.
     *
     * <p> The iterator returned by this method first yields all of the
     * elements of the provider cache, in instantiation order.  It then lazily
     * loads and instantiates any remaining providers, adding each one to the
     * cache in turn.
     *
     * <p> To achieve laziness the actual work of parsing the available
     * provider-configuration files and instantiating providers must be done by
     * the iterator itself.  Its {@link java.util.Iterator#hasNext hasNext} and
     * {@link java.util.Iterator#next next} methods can therefore throw a
     * {@link ServiceConfigurationError} if a provider-configuration file
     * violates the specified format, or if it names a provider class that
     * cannot be found and instantiated, or if the result of instantiating the
     * class is not assignable to the service type, or if any other kind of
     * exception or error is thrown as the next provider is located and
     * instantiated.  To write robust code it is only necessary to catch {@link
     * ServiceConfigurationError} when using a service iterator.
     *
     * <p> If such an error is thrown then subsequent invocations of the
     * iterator will make a best effort to locate and instantiate the next
     * available provider, but in general such recovery cannot be guaranteed.
     *
     * <blockquote style="font-size: smaller; line-height: 1.2"><span
     * style="padding-right: 1em; font-weight: bold">Design Note</span>
     * Throwing an error in these cases may seem extreme.  The rationale for
     * this behavior is that a malformed provider-configuration file, like a
     * malformed class file, indicates a serious problem with the way the Java
     * virtual machine is configured or is being used.  As such it is
     * preferable to throw an error rather than try to recover or, even worse,
     * fail silently.</blockquote>
     *
     * <p> The iterator returned by this method does not support removal.
     * Invoking its {@link java.util.Iterator#remove() remove} method will
     * cause an {@link UnsupportedOperationException} to be thrown.
     *
     * @return  An iterator that lazily loads providers for this loader's
     *          service
     */
    public Iterator<S> iterator() {
        return new Iterator<S>() {

            Iterator<Map.Entry<String,S>> knownProviders
                = providers.entrySet().iterator();

            public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
			    // 在reload方法中， lookupIterator = new LazyIterator(service, loader);

                return lookupIterator.hasNext();
            }

            public S next() {
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

        };
    }

{% endcodeblock %}



{% codeblock 解析SPI配置文件的方法 lang:java %}

    // Parse a single line from the given configuration file, adding the name
    // on the line to the names list.
    //
    private int parseLine(Class service, URL u, BufferedReader r, int lc,
                          List<String> names)
        throws IOException, ServiceConfigurationError
    {
        String ln = r.readLine();
        if (ln == null) {
            return -1;
        }
        int ci = ln.indexOf('#');
        if (ci >= 0) ln = ln.substring(0, ci);
        ln = ln.trim();
        int n = ln.length();
        if (n != 0) {
            if ((ln.indexOf(' ') >= 0) || (ln.indexOf('\t') >= 0))
                fail(service, u, lc, "Illegal configuration-file syntax");
            int cp = ln.codePointAt(0);
            if (!Character.isJavaIdentifierStart(cp))
                fail(service, u, lc, "Illegal provider-class name: " + ln);
            for (int i = Character.charCount(cp); i < n; i += Character.charCount(cp)) {
                cp = ln.codePointAt(i);
                if (!Character.isJavaIdentifierPart(cp) && (cp != '.'))
                    fail(service, u, lc, "Illegal provider-class name: " + ln);
            }
            if (!providers.containsKey(ln) && !names.contains(ln))
				//  添加当前行解析出的类名
                names.add(ln);
        }
        return lc + 1;
    }

    // Parse the content of the given URL as a provider-configuration file.
    //
    // @param  service
    //         The service type for which providers are being sought;
    //         used to construct error detail strings
    //
    // @param  u
    //         The URL naming the configuration file to be parsed
    //
    // @return A (possibly empty) iterator that will yield the provider-class
    //         names in the given configuration file that are not yet members
    //         of the returned set
    //
    // @throws ServiceConfigurationError
    //         If an I/O error occurs while reading from the given URL, or
    //         if a configuration-file format error is detected
    //
    private Iterator<String> parse(Class service, URL u)
        throws ServiceConfigurationError
    {
        InputStream in = null;
        BufferedReader r = null;
        ArrayList<String> names = new ArrayList<String>();
        try {
            in = u.openStream();
            r = new BufferedReader(new InputStreamReader(in, "utf-8"));
            int lc = 1;
            while ((lc = parseLine(service, u, r, lc, names)) >= 0);
        } catch (IOException x) {
            fail(service, "Error reading configuration file", x);
        } finally {
            try {
                if (r != null) r.close();
                if (in != null) in.close();
            } catch (IOException y) {
                fail(service, "Error closing configuration file", y);
            }
        }
        return names.iterator();
    }

{% endcodeblock %}


{% codeblock 默认情况下的DefaultSelectorProvider lang:java %}

public class DefaultSelectorProvider {

    /**
     * Prevent instantiation.
     */
    private DefaultSelectorProvider() { }

    /**
     * Returns the default SelectorProvider.
     */
    public static SelectorProvider create() {
        String osname = AccessController.doPrivileged(
            new GetPropertyAction("os.name"));
        if ("SunOS".equals(osname)) {
            return new sun.nio.ch.DevPollSelectorProvider();
        }

        // use EPollSelectorProvider for Linux kernels >= 2.6
        if ("Linux".equals(osname)) {
            String osversion = AccessController.doPrivileged(
                new GetPropertyAction("os.version"));
            String[] vers = osversion.split("\\.", 0);
            if (vers.length >= 2) {
                try {
                    int major = Integer.parseInt(vers[0]);
                    int minor = Integer.parseInt(vers[1]);
                    if (major > 2 || (major == 2 && minor >= 6)) {
                        return new sun.nio.ch.EPollSelectorProvider();
                    }
                } catch (NumberFormatException x) {
                    // format not recognized
                }
            }
        }

        return new sun.nio.ch.PollSelectorProvider();
    }

}

{% endcodeblock %}
