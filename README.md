## 前言
proxysql 是常用的MySQL中间件，一般用来做sql转发，改写，负载均衡等功用，今天第一篇，来解析一下proxysql的启动逻辑。

## 代码解析
入口函数在proxysql/src/main.cpp
```c++
int main(int argc, const char * argv[]) {

	{
    // 初始化一个MYSQL
		MYSQL *my = mysql_init(NULL);
		mysql_close(my);
//		cpu_timer t;
    // ProxySQL_Main_init： 主要配置在debug模式下全局参数,代码如下
    // #ifdef DEBUG
	  // GloVars.global.gdbg=false; 如果在DEBUG模式下。将全局变量gdbg设置为false
	  // glovars.has_debug=true;    如果在DEBUG模式下。has_debug设置为true
    // #else
	  // glovars.has_debug=false;    否则不开启debug模式
    // #endif /* DEBUG */
    //	// __thr_sfp=l_mem_init();
	  // proxysql_init_debug_prometheus_metrics(); 这里主要是初始化了一个prometheus::Family<prometheus::Counter>*的数组的map

		ProxySQL_Main_init();
#ifdef DEBUG
//		std::cerr << "Main init phase0 completed in ";
#endif
	}
	{
		cpu_timer t; // 初始化时间结构
    // ProxySQL_Main_process_global_variables中设置各种全局变量，没有设置配置文件，会自动加载proxysql.cfg默认配置文件
		ProxySQL_Main_process_global_variables(argc, argv);
		GloVars.global.start_time=monotonic_time(); // always initialize it 配置启动时间，实际上是调用了clock_gettime()函数来做的 
		srand(GloVars.global.start_time*thread_id());
		randID = rand();
#ifdef DEBUG
		std::cerr << "Main init global variables completed in ";
#endif
	}

  // 这部分主要是检查nlimit是否正常，通常来说，你需要设置一个合适的值，如65536，设置为最大。如果使用默认值会启动不起来
	struct rlimit nlimit;
	{
		int rc = getrlimit(RLIMIT_NOFILE, &nlimit);
		if (rc == 0) {
			proxy_info("Current RLIMIT_NOFILE: %lu\n", nlimit.rlim_cur);
			if (nlimit.rlim_cur <= 1024) {
				proxy_error("Current RLIMIT_NOFILE is very low: %lu .  Tune RLIMIT_NOFILE correctly before running ProxySQL\n", nlimit.rlim_cur);
				if (nlimit.rlim_max > nlimit.rlim_cur) {
					if (nlimit.rlim_max >= 102400) {
						nlimit.rlim_cur = 102400;
					} else {
						nlimit.rlim_cur = nlimit.rlim_max;
					}
					proxy_warning("Automatically setting RLIMIT_NOFILE to %lu\n", nlimit.rlim_cur);
					rc = setrlimit(RLIMIT_NOFILE, &nlimit);
					if (rc) {
						proxy_error("Unable to increase RLIMIT_NOFILE: %s: \n", strerror(errno));
					}
				} else {
					proxy_error("Unable to increase RLIMIT_NOFILE because rlim_max is low: %lu\n", nlimit.rlim_max);
				}
			}
		} else {
			proxy_error("Call to getrlimit failed: %s\n", strerror(errno));
		}
	}

  // 这里初始化了一个bootstrap_info_t的一个结构体，里面记录了proxysql它自身后端mysql数据库的信息，如版本、用户、密码、MYSQL_RES*结果集等
	bootstrap_info_t bootstrap_info {};
	// Try to connect to MySQL for performing the bootstrapping process:
	//   - If data isn't found we perform the bootstrap process.
	//   - If non-empty datadir is present, reconfiguration should be performed.
    // 以下部分，是初始化引导启动的部分，在初次运行时会走这部分逻辑。GloVars.global.gr_bootstrap_mode: 0
	if (GloVars.global.gr_bootstrap_mode == 1) {
		// Check the other required arguments for performing the bootstrapping process:
		//  - Username
		//  - Password - asked by prompt or supplied
		//  - Connection string parsing is required
		const string conn_uri { GloVars.global.gr_bootstrap_uri };
		const pair<int32_t,conn_uri_t> parse_uri_res { parse_conn_uri(conn_uri) };
		const conn_uri_t uri_data = parse_uri_res.second;

		if (parse_uri_res.first == EXIT_FAILURE) {
			proxy_info("Aborting bootstrap due to failed to parse or match URI - `%s`\n", to_string(uri_data).c_str());
			exit(parse_uri_res.first);
		} else {
			proxy_info("Bootstrap connection data supplied via URI - `%s`\n", to_string(uri_data).c_str());
		}

		const char* c_host = uri_data.host.c_str();
		const char* c_user = uri_data.user.empty() ? "root" : uri_data.user.c_str();
		const char* c_pass = nullptr;
		uint32_t port = uri_data.port == 0 ? 3306 : uri_data.port;
		uint32_t flags = CLIENT_SSL;

		nlohmann::ordered_json conn_data { { "host", c_host }, { "user", c_user }, { "port", port } };
		proxy_info("Performing bootstrap connection using URI data and defaults - `%s`\n", conn_data.dump().c_str());

		if (uri_data.pass.empty()) {
			c_pass = getpass("Enter password: ");
		} else {
			c_pass = uri_data.pass.c_str();
		}

		MYSQL* mysql = mysql_init(NULL);

		// SSL explicitly disabled by user for backend connections
		if (GloVars.global.gr_bootstrap_ssl_mode) {
			if (!strcasecmp(GloVars.global.gr_bootstrap_ssl_mode, "DISABLED")) {
				flags = 0;
			}
		}

		if (flags == CLIENT_SSL) {
			mysql_ssl_set(
				mysql,
				GloVars.global.gr_bootstrap_ssl_key,
				GloVars.global.gr_bootstrap_ssl_cert,
				GloVars.global.gr_bootstrap_ssl_ca,
				GloVars.global.gr_bootstrap_ssl_capath,
				GloVars.global.gr_bootstrap_ssl_cipher
			);
		}

		if (!mysql_real_connect(mysql, c_host, c_user, c_pass, nullptr, port, NULL, flags)) {
			proxy_error("Bootstrap failed, connection error '%s'\n", mysql_error(mysql));
			exit(EXIT_FAILURE);
		}

		if (uri_data.pass.empty()) {
			uint32_t passlen = strlen(c_pass);
			memset(static_cast<void*>(const_cast<char*>(c_pass)), 0, passlen);
		}

		// Get server default collation and version directly from initial handshake
		bootstrap_info.server_language = mysql->server_language;
		bootstrap_info.server_version = mysql->server_version;

		// Fetch all required data for Bootstrap
        // select * from performance_schema.replication_group
		int myrc = mysql_query(mysql, BOOTSTRAP_SELECT_SERVERS);

		if (myrc) {
			proxy_error("Bootstrap failed, query failed with error - %s\n", mysql_error(mysql));
			exit(EXIT_FAILURE);
		}

		MYSQL_RES* myres_members = mysql_store_result(mysql);

		if (myres_members == nullptr || mysql_num_rows(myres_members) == 0) {
			proxy_error("Bootstrap failed, expected server %s:%d to have Group Replication configured\n", c_host, port);
			exit(EXIT_FAILURE);
		}

		myrc = mysql_query(mysql, BOOTSTRAP_SELECT_USERS);

		if (myrc) {
			proxy_error("Bootstrap failed, query failed with error - %s\n", mysql_error(mysql));
			exit(EXIT_FAILURE);
		}

		MYSQL_RES* myres_users = mysql_store_result(mysql);

		if (myres_users == nullptr) {
			proxy_error("Bootstrap failed, storing resultset failed with error - %s\n", mysql_error(mysql));
			exit(EXIT_FAILURE);
		}

		// TODO-NOTE: Maybe further data verification should be performed here; bootstrap-info holding final types
		bootstrap_info.servers = myres_members;
		bootstrap_info.users = myres_users;

		// Setup a bootstrap account - monitoring
		const string account_create_policy {
			GloVars.global.gr_bootstrap_account_create == nullptr ? "if-not-exists" :
				GloVars.global.gr_bootstrap_account_create
		};

		if (GloVars.global.gr_bootstrap_account == nullptr && GloVars.global.gr_bootstrap_account_create != nullptr) {
			proxy_error("Bootstrap failed, option '--account-create' can only be used in combination with '--account'\n");
			exit(EXIT_FAILURE);
		}

		const uint32_t password_retries = GloVars.global.gr_bootstrap_password_retries;
		string new_mon_user {};
		string new_mon_pass {};

		if (GloVars.global.gr_bootstrap_account != nullptr) {
			const vector<string> valid_policies { "if-not-exists", "always", "never" };
			if (std::find(valid_policies.begin(), valid_policies.end(), account_create_policy) == valid_policies.end()) {
				proxy_error("Bootstrap failed, unknown '--account-create' option '%s'\n", account_create_policy.c_str());
				exit(EXIT_FAILURE);
			}

			// Since an account has been specified, we require the password for the account
			const string mon_user { GloVars.global.gr_bootstrap_account };
			const string get_acc_pass_msg { "Please enter MySQL password for " + mon_user + ": " };

			// Get the account pass directly from user input
			const string mon_pass = getpass(get_acc_pass_msg.c_str());

			// 1. Check if account exists
			const string get_user_cnt { "SELECT COUNT(*) FROM mysql.user WHERE user='" + mon_user + "'" };
			int cnt_err = mysql_query(mysql, get_user_cnt.c_str());
			MYSQL_RES* myres = mysql_store_result(mysql);

			if (cnt_err || myres == nullptr) {
				proxy_error("Bootstrap failed, detecting count existence failed with error - %s\n", mysql_error(mysql));
				exit(EXIT_FAILURE);
			}

			MYSQL_ROW myrow = mysql_fetch_row(myres);
			uint32_t acc_exists = atoi(myrow[0]);
			mysql_free_result(myres);

			if (account_create_policy == "if-not-exists") {
				// 2. Account doesn't exists, create new account. Otherwise reuse current
				if (acc_exists == 0) {
					pair<int32_t,acct_creds_t> new_creds { create_bootstrap_account(mysql, mon_user, mon_pass) };

					if (new_creds.first) {
						proxy_error("Bootstrap failed, user creation failed with error - %s\n", mysql_error(mysql));
						exit(EXIT_FAILURE);
					} {
						// Store the credentials as the new 'monitor' ones.
						new_mon_user = mon_user;
						new_mon_pass = new_creds.second.pass;
					}
				} else {
					new_mon_user = mon_user;
					new_mon_pass = mon_pass;
				}
			} else if (account_create_policy == "always") {
				if (acc_exists == 0) {
					pair<int32_t,acct_creds_t> new_creds { create_bootstrap_account(mysql, mon_user, mon_pass) };

					if (new_creds.first) {
						proxy_error("Bootstrap failed, user creation failed with error - %s\n", mysql_error(mysql));
						exit(EXIT_FAILURE);
					}
				} else {
					proxy_error(
						"Bootstrap failed, account '%s' already exists but supplied option '--account-create=\"always\"'\n",
						mon_user.c_str()
					);
					exit(EXIT_FAILURE);
				}

				new_mon_user = mon_user;
				new_mon_pass = mon_pass;
			} else if (account_create_policy == "never") {
				if (acc_exists == 0) {
					proxy_error(
						"Bootstrap failed, account '%s' doesn't exists but supplied option '--account-create=\"never\"'\n",
						mon_user.c_str()
					);
					exit(EXIT_FAILURE);
				}

				new_mon_user = mon_user;
				new_mon_pass = mon_pass;
			} else {
				proxy_error("Bootstrap failed, unknown '--account-create' option '%s'\n", account_create_policy.c_str());
				exit(EXIT_FAILURE);
			}
		} else {
			string prev_bootstrap_user {};
			string prev_bootstrap_pass {};

			if (Proxy_file_exists(GloVars.admindb)) {
				SQLite3DB::LoadPlugin(GloVars.sqlite3_plugin);
				SQLite3DB* configdb = new SQLite3DB();
				configdb->open((char*)GloVars.admindb, SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_FULLMUTEX);

				{
					const char check_table[] {
						"SELECT count(*) FROM sqlite_master WHERE type='table' AND name='bootstrap_variables'"
					};

					int table_exists = 0;

					char* error = nullptr;
					int cols = 0;
					int affected_rows = 0;
					SQLite3_result* resultset = NULL;

					configdb->execute_statement(check_table, &error , &cols , &affected_rows , &resultset);

					if (error == nullptr && resultset) {
						table_exists = atoi(resultset->rows[0]->fields[0]);
						delete resultset;
					} else {
						const char* err_msg = error != nullptr ? error : "Empty resultset";
						proxy_error("Bootstrap failed, query failed with error - %s", err_msg);
						exit(EXIT_FAILURE);
					}

					if (table_exists != 0) {
						const char query_user_pass[] {
							"SELECT variable_name,variable_value FROM bootstrap_variables"
								" WHERE variable_name='bootstrap_username' OR variable_name='bootstrap_password'"
								" ORDER BY variable_name"
						};
						configdb->execute_statement(query_user_pass, &error, &cols, &affected_rows, &resultset);

						if (resultset->rows.size() != 0) {
							prev_bootstrap_user = resultset->rows[1]->fields[1];
							prev_bootstrap_pass = resultset->rows[0]->fields[1];
						}

						if (resultset) {
							delete resultset;
						}
					}
				}

				delete configdb;
			}

			if (!prev_bootstrap_pass.empty() && !prev_bootstrap_user.empty()) {
				proxy_info(
					"Bootstrap info, detected previous bootstrap user '%s' reusing account...\n",
					prev_bootstrap_user.c_str()
				);

				new_mon_user = prev_bootstrap_user;
				new_mon_pass = prev_bootstrap_pass;
			} else {
				// Create random account with random password
				pair<int32_t,acct_creds_t> mon_creds { create_random_bootstrap_account(mysql, password_retries) };

				if (mon_creds.first) {
					proxy_error(
						"Bootstrap failed, user creation '%s' failed with error - %s\n",
						mon_creds.second.user.c_str(), mysql_error(mysql)
					);
					exit(EXIT_FAILURE);
				} else {
					new_mon_user = mon_creds.second.user;
					new_mon_pass = mon_creds.second.pass;
					bootstrap_info.rand_gen_user = true;
				}
			}
		}

		bootstrap_info.mon_user = new_mon_user;
		bootstrap_info.mon_pass = new_mon_pass;

		mysql_close(mysql);
	}

	{
		cpu_timer t;
		ProxySQL_Main_init_SSL_module();
#ifdef DEBUG
		std::cerr << "Main SSL init variables completed in ";
#endif
	}

	{
		cpu_timer t;
		int fd = -1;
		char buff[PATH_MAX+1];
		ssize_t len = -1;
#if defined(__FreeBSD__)
		len = readlink("/proc/curproc/file", buff, sizeof(buff)-1);
#else
		len = readlink("/proc/self/exe", buff, sizeof(buff)-1);
        // buff: /root/proxysql/src/proxysql 返回的是可执行文件的路径
//        printf("buff---> : %s \n", buff);
#endif
		if (len != -1) {
			buff[len] = '\0';
			fd = open(buff, O_RDONLY);
		}
		if(fd >= 0) {
			struct stat statbuf;
			if(fstat(fd, &statbuf) == 0) {
                // st_size： 文件实际大小
				unsigned char *fb = (unsigned char *)mmap(0, statbuf.st_size, PROT_READ, MAP_SHARED, fd, 0);
                if (fb != MAP_FAILED) {
                    // 文件加密？
					unsigned char temp[SHA_DIGEST_LENGTH];
                    // 计算HASH值
					SHA1(fb, statbuf.st_size, temp);
					binary_sha1 = (char *)malloc(SHA_DIGEST_LENGTH*2+1);
					memset(binary_sha1, 0, SHA_DIGEST_LENGTH*2+1);
					char buf[SHA_DIGEST_LENGTH*2 + 1];
					for (int i=0; i < SHA_DIGEST_LENGTH; i++) {
						sprintf((char*)&(buf[i*2]), "%02x", temp[i]);
					}
                    // 计算后的HASH值，写入内存中
					memcpy(binary_sha1, buf, SHA_DIGEST_LENGTH*2);
					munmap(fb,statbuf.st_size);
				} else {
					proxy_error("Unable to mmap %s: %s\n", buff, strerror(errno));
				}
			} else {
				proxy_error("Unable to fstat %s: %s\n", buff, strerror(errno));
			}
		} else {
			proxy_error("Unable to open %s: %s\n", argv[0], strerror(errno));
		}
#ifdef DEBUG
		std::cerr << "SHA1 generated in ";
#endif
	}
	if (GloVars.global.foreground==false) {
		{
			cpu_timer t;
      // 这个函数很重要，是启动守护进程的第一步，用到了libdaemon库，调用了daemon_retval_init()来初始化守护进程。后续还有个ProxySQL_daemonize_phase2
			ProxySQL_daemonize_phase1((char *)argv[0]);
#ifdef DEBUG
			std::cerr << "Main daemonize phase1 completed in ";
#endif
		}
	/* Do the fork */
    // daemon_fork(): c lib: libdaemon to fork process of daemon for proxysql
		if ((pid = daemon_fork()) < 0) {
			/* Exit on error */
            // Frees the resources allocated by daemon_retval_init().
			daemon_retval_done();
			exit(EXIT_FAILURE);

		} else if (pid) { /* The parent */

			ProxySQL_daemonize_wait_daemon();

		} else { /* The daemon */

			cpu_timer t;
			GloVars.global.start_time=monotonic_time();
            // signal监控事件
			GloVars.install_signal_handler();
            // ProxySQL_daemonize_phase2（）： Successfully started
			if (ProxySQL_daemonize_phase2()==false) {
				goto finish;
			}

#ifdef DEBUG
			std::cerr << "Main daemonize phase1 completed in ";
#endif
		}

		if (glovars.proxy_restart_on_error) {
			handleProcessRestart();
		}

	} else {
		GloAdmin->flush_error_log();
		GloVars.install_signal_handler();
	}

__start_label:
	{
		cpu_timer t;
		ProxySQL_Main_init_phase2___not_started(bootstrap_info);
#ifdef DEBUG
		std::cerr << "Main init phase2 completed in ";
#endif
	}
	if (glovars.shutdown) {
		goto __shutdown;
	}

	{
		cpu_timer t;
		ProxySQL_Main_init_phase3___start_all();
#ifdef DEBUG
		std::cerr << "Main init phase3 completed in ";
#endif
	}
#ifdef DEBUG
		std::cerr << "WARNING: this is a DEBUG release and can be slow or perform poorly. Do not use it in production" << std::endl;
#endif
	proxy_info("For information about products and services visit: https://proxysql.com/\n");
	proxy_info("For online documentation visit: https://proxysql.com/documentation/\n");
	proxy_info("For support visit: https://proxysql.com/services/support/\n");
	proxy_info("For consultancy visit: https://proxysql.com/services/consulting/\n");

	{
#if 0
		{
			// the following commented code is here only to manually test handleProcessRestart()
			// DO NOT ENABLE
			//proxy_info("Service is up\n");
			//sleep(2);
			//assert(0);
		}
#endif
		unsigned int missed_heartbeats = 0;
		unsigned long long previous_time = monotonic_time();
		unsigned int inner_loops = 0;
		unsigned long long time_next_version_check = 0;
		while (glovars.shutdown==0) {
			usleep(200000);
			if (disable_watchdog) {
				continue;
			}
			unsigned long long curtime = monotonic_time();
			if (GloVars.global.version_check) {
				if (curtime > time_next_version_check) {
					pthread_attr_t attr;
					pthread_attr_init(&attr);
					pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
					pthread_t thr;
					if (pthread_create(&thr, &attr, main_check_latest_version_thread, NULL) !=0 ) {
						perror("Thread creation");
						exit(EXIT_FAILURE);
					}
					if (time_next_version_check == 0)
						time_next_version_check = curtime;
					unsigned long long inter = 24*3600*1000;
					inter *= 1000;
					time_next_version_check += inter;
				}
			}
			inner_loops++;
			if (curtime >= inner_loops*300000 + previous_time ) {
				// if this happens, it means that this very simple loop is blocked
				// probably we are running under gdb
				previous_time = curtime;
				inner_loops = 0;
				continue;
			}
			if (GloMTH) {
				unsigned long long atomic_curtime = 0;
				unsigned long long poll_timeout = (unsigned int)GloMTH->variables.poll_timeout;
				unsigned int threads_missing_heartbeat = 0;
				poll_timeout += 1000; // add 1 second (rounding up)
				poll_timeout *= 1000; // convert to us
				if (curtime < previous_time + poll_timeout) {
					continue;
				}
				previous_time = curtime;
				inner_loops = 0;
				unsigned int i;
				if (GloMTH->mysql_threads) {
					for (i=0; i<GloMTH->num_threads; i++) {
						if (GloMTH->mysql_threads[i].worker) {
							atomic_curtime = GloMTH->mysql_threads[i].worker->atomic_curtime;
							if (curtime > atomic_curtime + poll_timeout) {
								threads_missing_heartbeat++;
							}
						}
					}
				}
#ifdef IDLE_THREADS
				if (GloVars.global.idle_threads) {
					if (GloMTH->mysql_threads) {
						for (i=0; i<GloMTH->num_threads; i++) {
							if (GloMTH->mysql_threads_idles[i].worker) {
								atomic_curtime = GloMTH->mysql_threads_idles[i].worker->atomic_curtime;
								if (curtime > atomic_curtime + poll_timeout) {
									threads_missing_heartbeat++;
								}
							}
						}
					}
				}
#endif
				if (threads_missing_heartbeat) {
					proxy_error("Watchdog: %u threads missed a heartbeat\n", threads_missing_heartbeat);
					missed_heartbeats++;
					if (missed_heartbeats >= (unsigned int)GloVars.restart_on_missing_heartbeats) {
#ifdef RUNNING_ON_VALGRIND
						proxy_error("Watchdog: reached %u missed heartbeats. Not aborting because running under Valgrind\n", missed_heartbeats);
#else
						if (GloVars.restart_on_missing_heartbeats) {
							proxy_error("Watchdog: reached %u missed heartbeats. Aborting!\n", missed_heartbeats);
							proxy_error("Watchdog: see details at https://github.com/sysown/proxysql/wiki/Watchdog\n");
							assert(0);
						}
#endif
					}
				} else {
					missed_heartbeats = 0;
				}
			}
		}
	}

__shutdown:

	proxy_info("Starting shutdown...\n");

	// First shutdown step is to unload plugins
	UnloadPlugins();

	ProxySQL_Main_init_phase4___shutdown();

	proxy_info("Shutdown completed!\n");

	if (glovars.reload) {
		if (glovars.reload==2) {
			GloVars.global.nostart=true;
		}
		glovars.reload=0;
		glovars.shutdown=0;
		goto __start_label;
	}

finish:
	//daemon_log(LOG_INFO, "Exiting...");
	proxy_info("Exiting...\n");
	daemon_retval_send(255);
	daemon_signal_done();
	daemon_pid_file_remove();

//	l_mem_destroy(__thr_sfp);

#ifdef RUNNING_ON_VALGRIND
	if (RUNNING_ON_VALGRIND==0) {
		if (__web_interface) {
			dlclose(__web_interface);
		}
		if (__mysql_ldap_auth) {
			dlclose(__mysql_ldap_auth);
		}
	}
#endif
	return 0;
}



```
