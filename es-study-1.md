# ES-Study
首先从https://github.com/elastic/elasticsearch上下载源码，我下载的是版本号为2.2的。
源码下载之后迫不及待的想知道程序的入口在哪里，那我们是不是可以通过启动脚本入口。然后进入es的目录/bin/找到es的启动脚本elasticsearch.sh。
在脚本的最后我们发现org.elasticsearch.bootstrap.Elasticsearch有这行代码，就推测这是ES的入口方法，然后我们代开下载好的源码进行查看。
发现Elasticsearch这个类很简单，在main函数中调用Bootstrap.init(args)。那我们接下来查看Bootstrap这个类相应的方法。
然后我们深入到Bootstrap的init方法中：
static void init(String[] args) throws Throwable {
        // Set the system property before anything has a chance to trigger its use
        System.setProperty("es.logger.prefix", "");

//目测下面这段逻辑是对环境的基础配置
        BootstrapCLIParser bootstrapCLIParser = new BootstrapCLIParser();
        CliTool.ExitStatus status = bootstrapCLIParser.execute(args);

        if (CliTool.ExitStatus.OK != status) {
            exit(status.status());
        }
//申明Bootstrap 的对象 是个线程安全的
        INSTANCE = new Bootstrap();

        boolean foreground = !"false".equals(System.getProperty("es.foreground", System.getProperty("es-foreground")));
        // handle the wrapper system property, if its a service, don't run as a service
        if (System.getProperty("wrapper.service", "XXX").equalsIgnoreCase("true")) {
            foreground = false;
        }

        Environment environment = initialSettings(foreground);
        Settings settings = environment.settings();
        setupLogging(settings, environment);
        checkForCustomConfFile();

        if (environment.pidFile() != null) {
            PidFile.create(environment.pidFile(), true);
        }

        if (System.getProperty("es.max-open-files", "false").equals("true")) {
            ESLogger logger = Loggers.getLogger(Bootstrap.class);
            logger.info("max_open_files [{}]", ProcessProbe.getInstance().getMaxFileDescriptorCount());
        }

        // warn if running using the client VM
        if (JvmInfo.jvmInfo().getVmName().toLowerCase(Locale.ROOT).contains("client")) {
            ESLogger logger = Loggers.getLogger(Bootstrap.class);
            logger.warn("jvm uses the client vm, make sure to run `java` with the server vm for best performance by adding `-server` to the command line");
        }

        try {
            if (!foreground) {
                Loggers.disableConsoleLogging();
                closeSystOut();
            }

            // fail if using broken version
            JVMCheck.check();
//根据参数设置启动项相关的参数
//我觉得这个方法最重要干的事就是
/*
node = nodeBuilder.build();
即初始化了Node对象
*/
            INSTANCE.setup(true, settings, environment);

            INSTANCE.start();

            if (!foreground) {
                closeSysError();
            }
        } catch (Throwable e) {
            // disable console logging, so user does not see the exception twice (jvm will show it already)
            if (foreground) {
                Loggers.disableConsoleLogging();
            }
            ESLogger logger = Loggers.getLogger(Bootstrap.class);
            if (INSTANCE.node != null) {
                logger = Loggers.getLogger(Bootstrap.class, INSTANCE.node.settings().get("name"));
            }
            // HACK, it sucks to do this, but we will run users out of disk space otherwise
            if (e instanceof CreationException) {
                // guice: log the shortened exc to the log file
                ByteArrayOutputStream os = new ByteArrayOutputStream();
                PrintStream ps = new PrintStream(os, false, "UTF-8");
                new StartupError(e).printStackTrace(ps);
                ps.flush();
                logger.error("Guice Exception: {}", os.toString("UTF-8"));
            } else {
                // full exception
                logger.error("Exception", e);
            }
            // re-enable it if appropriate, so they can see any logging during the shutdown process
            if (foreground) {
                Loggers.enableConsoleLogging();
            }
            
            throw e;
        }
    }
