  

## Docker解析
### 1.1 docker服务

一般意义上的docker，是指启动一个docker 进程，可以看到，其实用systemd启动是docker进程是dockerd 和docker proxy这两个二进制文件，其中dockerd指定的containerd 是另外单独安装的containerd，其实这些架构每个版本都有区别，比较早的版本docker是没有分containerd的，后面又把这部分给开源给cri社区，可以看到后面版本的安装步骤都不一样，下面的版本是1.19
```shell
docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-08-31 17:50:53 CST; 3min 9s ago
     Docs: https://docs.docker.com
 Main PID: 20656 (dockerd)
    Tasks: 26
   Memory: 51.1M
   CGroup: /system.slice/docker.service
           ├─20656 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
           └─21055 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0...
```


### 1.2 dockerd

1. 先看看dockerd 的entry point
  docker-ce/components/engine/cmd/dockerd/README.md提示:docker.go 是entry point

  ```go
  docker.go contains Docker daemon's main function.
  
  This file provides first line CLI argument parsing and environment variable setting.
  ```

  

​	一般来说entry point都是在cmd，遵循cmd/appname/main.go 的规范，然后appname的main.go 会被编译成appname的二进制，这也是看go项目源码的一个技巧
一个包下面只有一个main.所以也可以直接搜索main()

2. 核心就是构建一个命令，然后execute,所以需要看这个cmd 的run方法
   ```go
   cmd, err := newDaemonCommand()
   if err != nil {
   	onError(err)
   }
   cmd.SetOut(stdout)
   if err := cmd.Execute(); err != nil {
   	onError(err)
   }
   ```

3. 跳转不了，这种老项目都不是go module的，没办法搞掂依赖的话是没办法跳转的，查了下这种老项目，其实还是可以配置好的 

4. 下载源码，新建各个项目的地址

   ```shell
   mkdir -p /opt/docker-moby/src/github.com/docker
   mkdir -p /opt/docker-cli/src/github.com/docker
   mkdir -p /opt/docker-containerd/src/github.com/containerd
   mkdir -p /opt/docker-runc/src/github.com/opencontainers
   git clone https://github.com/moby/moby.git /opt/docker-moby/src/github.com/docker/docker
   git clone https://github.com/docker/cli.git /opt/docker-cli/src/github.com/docker/cli
   git clone https://github.com/containerd/containerd.git /opt/docker-containerd/src/github.com/containerd/containerd
   git clone https://github.com/opencontainers/runc.git /opt/docker-runc/src/github.com/opencontainers/runc
   //刚刚安装的git要配置下proxy
   git config --global url."https://gitclone.com/".insteadOf https://
   ```

   
5. 设置gopath环境变量

   ```shell
   export GOPATH="/opt/docker-moby:/opt/docker-containerd:/opt/docker-cli:/opt/docker-runc"
   ```

   

6. 打开code /opt/docker-moby/src/github.com/docker/docker如果是goland，需要取消掉gomodule 模式而且添加这几个path，好了现在可以随便点击跳转了，舒服。

   ```go
   cmd := &cobra.Command{
   		Use:           "dockerd [OPTIONS]",
   		Short:         "A self-sufficient runtime for containers.",
   		SilenceUsage:  true,
   		SilenceErrors: true,
   		Args:          cli.NoArgs,
   		RunE: func(cmd *cobra.Command, args []string) error {
   			opts.flags = cmd.Flags()
   			return runDaemon(opts)
   		},
   		DisableFlagsInUseLine: true,
   		Version:               fmt.Sprintf("%s, build %s", dockerversion.Version, dockerversion.GitCommit),
   	}
   ----
   //最重要的部分就是这里的start
   runDaemon():
   	err = daemonCli.start(opts)
   	notifyShutdown(err)
   	return err
   ```

   
7. daemon.go
    daemonCli.start 方法很长，先看看server的启动
    
    ```go
    func (cli *DaemonCli) start(opts *daemonOptions) (err error) {
        ...
    	// The serve API routine never exits unless an error occurs
    	// We need to start it as a goroutine and wait on it so
    	// daemon doesn't exit
    	serveAPIWait := make(chan error)
    	go cli.api.Wait(serveAPIWait)
    
    }
    
    //api/server/server.go
    // Wait blocks the server goroutine until it exits.
    // It sends an error message if there is any error during
    // the API execution.
    func (s *Server) Wait(waitChan chan error) {
    	if err := s.serveAPI(); err != nil {
    		logrus.Errorf("ServeAPI error: %v", err)
    		waitChan <- err
    		return
    	}
    	waitChan <- nil
    }
    
    ```
    
    
8. Http sever 部分差不多就这样了，其实大部分流程都差不多，最后都是用标准库net/http包去做实际的serve，核心还是看看handler的逻辑，router的注册这些

   ```go
   //api/server/server.go
   // serveAPI loops through all initialized servers and spawns goroutine
   // with Serve method for each. It sets createMux() as Handler also.
   func (s *Server) serveAPI() error {
   	var chErrors = make(chan error, len(s.servers))
   	for _, srv := range s.servers {
           //handler的逻辑是理解这个webserver功能的关键
   		srv.srv.Handler = s.routerSwapper
   		go func(srv *HTTPServer) {
   			var err error
   			logrus.Infof("API listen on %s", srv.l.Addr())
   			if err = srv.Serve(); err != nil && strings.Contains(err.Error(), "use of closed network connection") {
   				err = nil
   			}
   			chErrors <- err
   		}(srv)
   	}
   	for range s.servers {
   		err := <-chErrors
   		if err != nil {
   			return err
   		}
   	}
   	return nil
   }
   ```

   
9. initRouter():

   cmd/dockerd/daemon.go,被start()调用进行router的初始化，传入参数是opts，一个routeroptions类型

   

   ```go
   func initRouter(opts routerOptions) {
   	decoder := runconfig.ContainerDecoder{}
   
   	routers := []router.Router{
   		// we need to add the checkpoint router before the container router or the DELETE gets masked
   		checkpointrouter.NewRouter(opts.daemon, decoder),
   		container.NewRouter(opts.daemon, decoder),
   		image.NewRouter(opts.daemon.ImageService()),
   		systemrouter.NewRouter(opts.daemon, opts.cluster, opts.buildCache, opts.buildkit, opts.features),
   		volume.NewRouter(opts.daemon.VolumesService()),
   		build.NewRouter(opts.buildBackend, opts.daemon, opts.features),
   		sessionrouter.NewRouter(opts.sessionManager),
   		swarmrouter.NewRouter(opts.cluster),
   		pluginrouter.NewRouter(opts.daemon.PluginManager()),
   		distributionrouter.NewRouter(opts.daemon.ImageService()),
   	}
       ...
       opts.api.InitRouter(routers...)
   }
   
   // InitRouter initializes the list of routers for the server.
   // This method also enables the Go profiler.
   func (s *Server) InitRouter(routers ...router.Router) {
   	s.routers = append(s.routers, routers...)
   	
       //mux.Router是一个包含Handler ，router列表的一个结构体
   	m := s.createMux()
   	s.routerSwapper = &routerSwapper{
   		router: m,
   	}
   }
   //大致的调用链是:
   routers 初始化-->opts.api.InitRouter(routers...)-->routers 传给server，然后server用createMux注册、
   
   //最原始的Router的定义是个接口，有Router()方法，返回Route的列表
   // Route defines an individual API route in the docker server.
   // Router defines an interface to specify a group of routes to add to the docker server.
   type Router interface {
   	// Routes returns the list of routes to add to the docker server.
   	Routes() []Route
   }
   // Route defines an individual API route in the docker server.
   type Route interface {
   	// Handler returns the raw function to create the http handler.
   	Handler() httputils.APIFunc
   	// Method returns the http method that the route responds to.
   	Method() string
   	// Path returns the subpath where the route responds to.
   	Path() string
   }
   
   //看看实现，都在router文件夹下呢
   // buildRouter is a router to talk with the build controller
   type buildRouter struct {
   	backend  Backend
   	daemon   experimentalProvider
       //这个是实现的方法
   	routes   []router.Route
   	features *map[string]bool
   }
   
   ```

   ![image-20230902123812258](C:\Users\jimmy\AppData\Roaming\Typora\typora-user-images\image-20230902123812258.png)
10. mux.Router的实现，总的来说，这个Router是哪来绑定路由到handler 的，而且用到了装饰器模式

    ```go
    type Router struct {
    	// Configurable Handler to be used when no route matches.
    	NotFoundHandler http.Handler
    
    	// Configurable Handler to be used when the request method does not match the route.
    	MethodNotAllowedHandler http.Handler
    
    	// Routes to be matched, in order.
    	routes []*Route
    
    	// Routes by name for URL building.
    	namedRoutes map[string]*Route
    
    	// If true, do not clear the request context after handling the request.
    	//
    	// Deprecated: No effect, since the context is stored on the request itself.
    	KeepContext bool
    
    	// Slice of middlewares to be called after a match is found
    	middlewares []middleware
    
    	// configuration shared with `Route`
    	routeConf
    }
    
    //Path registers a new route with a matcher for the URL path.
    // See Route.Path().
    func (r *Router) Path(tpl string) *Route {
    	return r.NewRoute().Path(tpl)
    }
    ```

    
11. 看看实际注册的路由和handler方法吧，这个比较关键

    ```go
    //选这个看看
    func initRouter(opts routerOptions) {
    	decoder := runconfig.ContainerDecoder{}
    
    	routers := []router.Router{
    		// we need to add the checkpoint router before the container router or the DELETE gets masked
    		checkpointrouter.NewRouter(opts.daemon, decoder),
    ...
    }
    
    // NewRouter initializes a new checkpoint router
    func NewRouter(b Backend, decoder httputils.ContainerDecoder) router.Router {
    	r := &checkpointRouter{
    		backend: b,
    		decoder: decoder,
    	}
        //这里是关键
    	r.initRoutes()
    	return r
    }
    //第一个参数是path，第二个是handler
    func (r *checkpointRouter) initRoutes() {
    	r.routes = []router.Route{
    		router.NewGetRoute("/containers/{name:.*}/checkpoints", r.getContainerCheckpoints, router.Experimental),
    		router.NewPostRoute("/containers/{name:.*}/checkpoints", r.postContainerCheckpoint, router.Experimental),
    		router.NewDeleteRoute("/containers/{name}/checkpoints/{checkpoint}", r.deleteContainerCheckpoint, router.Experimental),
    	}
    }
    
    //看看handler，这里是理解请求处理逻辑的关键，其实看到这，可以看到前面一堆的初始化，如果用框架是会简化很多工作，gin的Route是做了很多工作
    func (s *checkpointRouter) getContainerCheckpoints(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
    	if err := httputils.ParseForm(r); err != nil {
    		return err
    	}
    
    	checkpoints, err := s.backend.CheckpointList(vars["name"], types.CheckpointListOptions{
    		CheckpointDir: r.Form.Get("dir"),
    	})
    
    	if err != nil {
    		return err
    	}
    
    	return httputils.WriteJSON(w, http.StatusOK, checkpoints)
    }
    ```

    

