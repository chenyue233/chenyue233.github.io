# Golang使用viper读取配置,修改实时生效


## Viper介绍
#### Viper是大佬在github开源的由golang编写的读取配置的解决方案。
#### github地址: <https://github.com/spf13/viper>
#### 支持从多种格式文件中读取配置，实时观看和重新读取配置，支持设置默认值、从环境变量读取、从命令行读取、从文件流中读取等。

#### 读取配置的优先级:
- 通过方法SET的配置值
- 命令行设置的配置值
- 环境变量
- 配置文件读取
- 默认值
  
## 实践:
#### 配置文件 settings.yml
``` yml
application:
  # dev开发环境 test测试环境 prod线上环境
  mode: dev
  # 服务器ip，默认使用 0.0.0.0
  host: 0.0.0.0
  # 服务名称
  name: test
  # 服务端口号
  port: 8088
database:
  # 数据库类型 mysql，sqlite3， postgres
  driver: mysql
  # 数据库连接字符串 mysql 缺省信息 charset=utf8&parseTime=True&loc=Local&timeout=1000ms
  source: root:123456@tcp(127.0.0.1:3306)/test?charset=utf8&parseTime=True&loc=Local&timeout=1000ms
redis:
  addr: 127.0.0.1:6379
  poolSize: 500
  password: "123456"
  db: 0
```

#### 读取配置文件代码

```golang
package config

const (
    // 指定配置文件后缀
    configFileType = "yml"
)

// 初始化全局config实例
var GlobalConfig = new(Config)

//Config 配置struct
type Config struct {
    Application `mapstructure:"application"`
    Database    `mapstructure:"database"`
    Redis       `mapstructure:"redis"`
}

//Application 应用基本信息
type Application struct {
    Host string `mapstructure:"host"`
    Port string `mapstructure:"port"`
    Name string `mapstructure:"name"`
    Mode string `mapstructure:"mode"`
}

//Database 数据库
type Database struct {
    Driver string `mapstructure:"driver"`
    Source string `mapstructure:"source"`
}
//Redis 缓存数据库
type Redis struct {
    Addr     string `mapstructure:"addr"`
    PoolSize int    `mapstructure:"poolSize"`
    Password string `mapstructure:"password"`
    DB       int    `mapstructure:"db"`
}


//Setup 配置初始化
func Setup(path string) {
    // 初始化 viper 配置
    viper.SetConfigFile(path)
    viper.SetConfigType(configFileType)
    if err := viper.ReadInConfig(); err != nil {
    panic(fmt.Sprintf("读取配置文件失败，请检查 settings.yml 配置文件是否存在: %v", err))
    }
    // 将读取的配置信息保存至全局变量Conf
    if err := viper.Unmarshal(GlobalConfig); err != nil {
    log.Printf("unmarshal conf failed, err:%s \n", err)
    }
    // 监听配置文件修改

    viper.WatchConfig()
    // 注意！！！配置文件发生变化后要同步到全局变量Conf
    viper.OnConfigChange(func(in fsnotify.Event) {
    fmt.Println("夭寿啦~有人正在修改配置文件")
    if err := viper.Unmarshal(GlobalConfig); err != nil {
      log.Printf("unmarshal conf failed, err:%s \n", err)
    }
    })

}
```

## 源码解读
#### viper.SetConfigFile(path)

``` golang
// SetConfigFile explicitly defines the path, name and extension of the config file.
// Viper will use this and not check any of the config paths.
func SetConfigFile(in string) { v.SetConfigFile(in) }
func (v *Viper) SetConfigFile(in string) {
  if in != "" {
  v.configFile = in
  }
}

```

接收一个string类型的参数,参数为配置文件存放的相对路径,将接受到的参数赋值给viper实例的configFile。  
如果配置文件settings.yml所在地址为project/settings.yml,直接传入setting.yml即可。  
如果配置文件所在的地址为project/config/settings.yml,则需要传入参数config/settings.yml。

## 源码解读
#### viper.SetConfigType(configFileType)

```golang

// SetConfigType sets the type of the configuration returned by the
// remote source, e.g. "json".
func SetConfigType(in string) { v.SetConfigType(in) }
func (v *Viper) SetConfigType(in string) {
  if in != "" {
  v.configType = in
  }
}
```

接受一个string类型的参数，参数为配置文件的类型，将接收到的参数赋值给viper实例的configType。

## 源码解读
#### viper.WatchConfig()

```golang

func WatchConfig() { v.WatchConfig() }

func (v *Viper) WatchConfig() {
  initWG := sync.WaitGroup{}
  initWG.Add(1)
  go func() {
  watcher, err := fsnotify.NewWatcher()
  if err != nil {
    log.Fatal(err)
  }
  defer watcher.Close()
  // we have to watch the entire directory to pick up renames/atomic saves in a cross-platform way
  filename, err := v.getConfigFile()
  if err != nil {
    log.Printf("error: %v\n", err)
    initWG.Done()
    return
  }

  configFile := filepath.Clean(filename)
  configDir, _ := filepath.Split(configFile)
  realConfigFile, _ := filepath.EvalSymlinks(filename)

  eventsWG := sync.WaitGroup{}
  eventsWG.Add(1)
  go func() {
	for {
		select {
		case event, ok := <-watcher.Events:
			if !ok { // 'Events' channel is closed
				eventsWG.Done()
				return
			}
			currentConfigFile, _ := filepath.EvalSymlinks(filename)
			// we only care about the config file with the following cases:
			// 1 - if the config file was modified or created
			// 2 - if the real path to the config file changed (eg: k8s ConfigMap replacement)
			const writeOrCreateMask = fsnotify.Write | fsnotify.Create
			if (filepath.Clean(event.Name) == configFile &&
				event.Op&writeOrCreateMask != 0) ||
				(currentConfigFile != "" && currentConfigFile != realConfigFile) {
				realConfigFile = currentConfigFile
				err := v.ReadInConfig()
				if err != nil {
					log.Printf("error reading config file: %v\n", err)
				}
				if v.onConfigChange != nil {
					v.onConfigChange(event)
				}
			} else if filepath.Clean(event.Name) == configFile &&
				event.Op&fsnotify.Remove&fsnotify.Remove != 0 {
				eventsWG.Done()
				return
			}

		case err, ok := <-watcher.Errors:
			if ok { // 'Errors' channel is not closed
				log.Printf("watcher error: %v\n", err)
			}
			eventsWG.Done()
			return
		}
	}
  }()
  watcher.Add(configDir)
  initWG.Done()   // done initializing the watch in this go routine, so the parent routine can move on...
  eventsWG.Wait() // now, wait for event loop to end in this go-routine...
  }()
  initWG.Wait() // make sure that the go routine above fully ended before returning
}

```
监听文件变为的WatchConfig 函数依赖第三方包:<https://github.com/fsnotify/fsnotify>,该包主要主要作用是监听文件变化，通知用户。 


```golang
    initWG := sync.WaitGroup{}
    initWG.Add(1)
    
```
   
  初始化一个协程等待组initWG，添加一个干活的协程小弟。


```golang
    watcher, err := fsnotify.NewWatcher()
		if err != nil {
			log.Fatal(err)
		}
		defer watcher.Close()
		// we have to watch the entire directory to pick up renames/atomic saves in a cross-platform way
		filename, err := v.getConfigFile()
		if err != nil {
			log.Printf("error: %v\n", err)
			initWG.Done()
			return
		}

		configFile := filepath.Clean(filename)
		configDir, _ := filepath.Split(configFile)
        realConfigFile, _ := filepath.EvalSymlinks(filename)

        eventsWG := sync.WaitGroup{}
        eventsWG.Add(1)
        go func() {
			for {
				select {
                case event, ok := <-watcher.Events:
                
                case err, ok := <-watcher.Errors:
                }    
```  

  在协程小弟中初始化fsnotify监听文件变化实例,该实例会等待文件的读取事件，如果I/O完成了读取任务，会将相应的修改或者创建事件注入到Event对象中。在协程小弟中再实例化一个协程等待组，在小弟协程中添加一个协程小弟弟，协程小弟弟使用select监听文件变化实例的两个chan，一个chan是文件变化事件及Event，另外一个是文件变化错误chan。  
  filepath.Clean 删除多余的路径分隔符，获取最短路径  
  filepath.Split 分隔目录与文件名
  filepath.EvalSymlinks 返回符号路径指向的路径名，可以直接访问

```golang

const writeOrCreateMask = fsnotify.Write | fsnotify.Create
if (filepath.Clean(event.Name) == configFile &&  event.Op&writeOrCreateMask != 0) ||
(currentConfigFile != "" && currentConfigFile != realConfigFile)

```
writeOrCreateMask 为监听文件修改实例的修改事件对应数字按位或的结果(修改事件 fsnotify.Write 对应数字为2，  修改事件 fsnotify.Create 对应数字为1 )writeOrCreateMask 结果为 11  
filepath.Clean(event.Name) == configFile 判断文件修改实例的文件名与配置文件名是否相等  
event.Op&writeOrCreateMask != 0  判断文件修改实例的事件数字与writeOrCreateMask 按位与的结果是否为0,因为writeOrCreateMask 是创建和修改数字按位或的结果11,只要实例的event.Op为修改或者创建其中一项及即1或者2，event.Op&writeOrCreateMask(01或者11与11按位与)就不可能为0.作用就是判断实例是否有修改或者创建的事件。  
currentConfigFile != "" && currentConfigFile != realConfigFile， currentConfigFile是监听文件变化实例监听的文件符号链接，realConfigFile是配置文件的符号链接。如果监听事件的符号链接不为空而且不与配置文件符号链接相等则说明修改前和修改后不是同一个文件。

符合以上条件了就说明配置文件发生了修改或者创建的变化，通过ReadInConfig重写加载配置到viper实例中。如果存在回调函数onConfigChange，就调用一次回调函数。

## 调用

### 为GlobalConfig 赋值
```golang
package main

func mian(){
  hpconf.Setup("config/settings.yml")  
}
```
在main函数调用setup函数为GlobalConfig赋值，就可以直接使用GlobalConfig读取配置了。


### 使用GlobalConfig 初始化redis配置

```golang
package cache
var RedisClient *redis.Client
import (
    "config"
)

// Setup redis连接初始化
func Setup() {
	Client := redis.NewClient(&redis.Options{
		Addr:         config.GlobalConfig.Addr,
		PoolSize:     config.GlobalConfig.PoolSize,
		Password:     config.GlobalConfig.Password,
		DB:           config.GlobalConfig.DB,
		ReadTimeout:  time.Millisecond * time.Duration(100),
		WriteTimeout: time.Millisecond * time.Duration(100),
		IdleTimeout:  time.Second * time.Duration(60),
	})

	if err := Client.Ping().Err(); err != nil {
		log.Fatal("connect to redis err:", err)
		return
	} else {
		log.Println("Database connect success")
	}
	RedisClient = Client
	return
}
···

