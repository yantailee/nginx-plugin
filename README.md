   一般情况下，当用户安装云锁的时候，云锁会自动适配nginx版本，使用我们已经预编译好的包含云锁模块的
nginx备份并替换掉您当前系统中使用的nginx。卸载时，会将系统原始nginx文件替换回来。因此，云锁可保护使用nginx搭建的网站，开创了这个领域的先河。

	我们秉承着让安全变得更简单的宗旨，使云锁尽可能多的适配广大用户的nginx版本（目前无法覆盖所有nginx版本），
决定开放云锁nginx模块的源码，让有能力的小伙伴们自己动手同云锁一起打造一个安全的nginx环境。

	此文档假设您曾经编译过nginx或tengine源代码，如果您之前没有经验，请参考：http://nginx.org/en/docs/configure.html
	

1. 如何编译云锁nginx模块
步骤如下：
    1) 为避免意外情况发生， 请先将系统当前使用中的nginx进行备份(包括相关的网站配置文件)
	2) wget https://codeload.github.com/yunsuo-open/nginx-plugin/zip/master -O nginx-plugin-master.zip
	3) unzip nginx-plugin-master.zip
	4) cd nginx-plugin-master
	5) pwd 获取当前云锁插件源码所在目录的全路径 (假设为：/home/nginx-plugin-master，实际情况以pwd输出为准)
	
	6) 如果您的环境是tengine,可以跳过这一步。对于nginx版本，由于其不支持post过滤，所以需要修改nginx源码目录下src/http/ngx_http_upstream.c 文件
        步骤如下：
		a. 查找 static void ngx_http_upstream_init_request(ngx_http_request_t *r)函数，在其
		所在行上方添加：int ngx_http_yunsuo_post_in_handler(ngx_http_request_t *r);
		
		b. 在ngx_http_upstream_init_request函数开头，变量声明后，添加：
			if(ngx_http_yunsuo_post_in_handler(r)) 
			{
				return;
			}
		
		什么？没看懂？好吧，以nginx-1.0.11为例：
		修改前源码：
			static void
			ngx_http_upstream_init_request(ngx_http_request_t *r)
			{
				ngx_str_t                      *host;
				ngx_uint_t                      i;
				ngx_resolver_ctx_t             *ctx, temp;
				ngx_http_cleanup_t             *cln;
				ngx_http_upstream_t            *u;
				ngx_http_core_loc_conf_t       *clcf;
				ngx_http_upstream_srv_conf_t   *uscf, **uscfp;
				ngx_http_upstream_main_conf_t  *umcf;
		
				if (r->aio) {
					return;
				}
		
				u = r->upstream;
				
				......
				
			}
		修改后源码：
			/*这段是添加的*/
			int ngx_http_yunsuo_post_in_handler(ngx_http_request_t *r);
			/*------------*/
	
			static void
			ngx_http_upstream_init_request(ngx_http_request_t *r)
			{
				ngx_str_t                      *host;
				ngx_uint_t                      i;
				ngx_resolver_ctx_t             *ctx, temp;
				ngx_http_cleanup_t             *cln;
				ngx_http_upstream_t            *u;
				ngx_http_core_loc_conf_t       *clcf;
				ngx_http_upstream_srv_conf_t   *uscf, **uscfp;
				ngx_http_upstream_main_conf_t  *umcf;
		
				/*这段是添加的*/
				if(ngx_http_yunsuo_post_in_handler(r)) 
				{
					return;
				}
				/*------------*/
				
				if (r->aio) {
					return;
				}
		
				u = r->upstream;
				
				......
				
			}
		
	7) 云锁的nginx插件模块是标准的nginx模块，所以您在编译nginx过程中，configure时只要添加额外参数
	   --add-module=/home/nginx-plugin-master(注意：/home/nginx-plugin-master为示例，实际路径以步骤5中pwd命令为准）
	   即可让nginx支持云锁的功能
	   
	   示例如下：
	   假设您之前configure时的命令如下：
		./configure --prefix=/usr/local/nginx --with-http_stub_status_module \
			--with-http_ssl_module --with-http_gzip_static_module \
			--add-module=../ngx_cache_purge-1.3 
	   现在的configure时的命令如下：
		./configure --prefix=/usr/local/nginx --with-http_stub_status_module \
			--with-http_ssl_module --with-http_gzip_static_module \
			--add-module=../ngx_cache_purge-1.3  --add-module=/home/nginx-plugin-master
			
	8) 编译 nginx (注意：如果原本已经有 nginx, 只执行 make 即可，make install 会覆盖掉你的 nginx.conf)
	
	9) 将系统当前使用中的nginx二进制文件替换为刚刚编译好的包含了云锁模块的nginx文件即可
	

2. 让云锁识别您自己编译的nginx
步骤如下：
	1)安装云锁，如果您已经安装了云锁，可跳过此步骤。如果还没有，请到http://www.yunsuo.com.cn/ht/software/
	  下载并安装云锁
	2) cd /usr/local/yunsuo_agent/nginx/
	3) ./configure_compiled_nginx nginx_install_path (nginx_install_path为nginx的安装路径,即configure时 --prefix=path
	   如果未指定过路径, 那么默认为/usr/local/nginx)
	   

FAQ：
1)什么情况下我需要自己编译云锁的nginx模块？
	a.当您的nginx使用了第三方或者自己开发的模块的时候，需要编译云锁的nginx模块。您可以通过nginx -V命令查看输出的
	 信息里是否包含了 --add-module= 的字样 (例如：--add-module=../ngx_cache_purge-1.3 说明使用了ngx_cache_purge-1.3第三方
	 模块)
	b.当使用tengine的时候，需要编译云锁的nginx模块
	c.当您发现当前使用的nginx版本比我们自动安装的版本高的时候，可以自己编译云锁的nginx模块

2)如果我把云锁卸载了，nginx需要重新编译吗？
      不需要，云锁的nginx模块会判断云锁是否安装，如果不安装则不生效。当然您也可以替换回之前的nginx

    3)我应该先安装云锁，还是先编译nginx?
  都可以，没有先后顺序关系


  

	   
