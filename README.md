####前言
   询价试驾等订单信息的有效性直接影响经销商的售车效果，如何保证线索信息的有效性，防止刷单等行为是下单流程中需要实现的一个重要模块
####业务分析
现实环境下的刷单行为主要包含以下两种

 - 针对特地商家的刷单行为：针对某个商家刷单，多为有竞争关系的经销商间发起的攻击，数据特点为商家某天的数据量远大高于历史均值
 - 无特定商家的刷单行为：多为黑客发起的随机攻击，数据特点是指定ip地址瞬间订单量较大或者某天数据量远大于历史均值
####解决方案
 - 对于集中发起下单行为的攻击，建立相应的计数器，当指定时间段的数据量超过阀值后是发出报警，本场景中为检测每个ip或商家单位时间的订单量是否超过阀值

 - 同时建立ip及商家每日订单数量的统计表，下单时计算当日的订单量是否超过了历史均值的指定倍数，此处有个缺点是对于统计表中相应数据较少无法获得正确的统计值的情况需要不过滤
####代码实现

 1,环境准备

 - 由于centos自带的git版本较低导致后续的工具安装有问题，第一步升级git
```bash
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
yum install gcc perl-ExtUtils-MakeMaker
yum remove git

cd /usr/src
wget https://www.kernel.org/pub/software/scm/git/git-2.0.4.tar.gz
tar xzf git-2.0.4.tar.gz

cd git-2.0.4
make prefix=/usr/local/git all
make prefix=/usr/local/git install
echo "export PATH=$PATH:/usr/local/git/bin" >> /etc/bashrc
source /etc/bashrc
```
 - 安装gvm用于管理本地的golang版本
```bash
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer) 
```
 - 安装golang及配置开发路径
```bash
#创建GOPATH对应的文件夹
mkdir ~/autohome/{bin,pkg,src}
gvm install go1.4 --source=https://github.com/golang/go
#安装交叉编译库
gvm cross linux amd64
#配置GOPATH
gvm pkgenv
#在打开的编辑器中修改GOPATH指向刚才创建的文件夹
gvm use go1.4 --default
#运行go env检查是否配置正确
go env
```
 - 安装开发依赖的第三方库
```bash
go get github.com/astaxie/beego
go get github.com/beego/bee
go get github.com/denisenkom/go-mssqldb
go get github.com/garyburd/redigo/redis
go get github.com/go-xorm/xorm
go get github.com/sdhjl2000/ecblib
#对于无法下载安装的库安装命令，然后使用rz命令从本地上传解压到src目录
yum install lrzsz
```
2,实现方法

 - 时间窗:以IP地址作为Key向Redis中累加请求次数,设置数据的过期时间为限定时间,当累加次数超过限定次数时进行报警，该方法的优点是节省存储,无需做数据同步,缺点是如果在一个时间窗周期内请求接近阀值的量然后等候到下一个时间周期再请求接近阀值的量，实际上两个时间窗中的一段时间请求是超过阀值的
```go
    rediskey := or.IPAddress + "TimeLimit"
	currentcount, errg := redis.Int(r.Do("GET", rediskey))
	if errg == nil {
		if currentcount > TimeThreshold {//当前时间窗内超过阀值
			return ResultMessage{ReturnCode: 1, Message: errmsg}
		}
	}
	newcount, _ := redis.Int(r.Do("incr", rediskey))//累加计数
	if newcount == 1 {
		r.Do("EXPIRE", rediskey, TimeLimit*60)//设置过期时间
	}
```
 - 流量窗：创建一个以IP为key,数据量阀值大小的队列为value的map，当有请求是向对应的queue插入一条当前时间的记录，当队列的长度大于限定大小时取出最前面的数据与当前时间比对，如果小于当前时间减时间阀值则报警,优点是时间窗是滑动的完全限定了任何时间段请求量不会超过阀值，确定是占用数据空间大，需要维护一个线程安全的map和queue保证数据的正确性，下面的代码使用buffer channel 作为线程安全的队列
```go
tchan, _ := ipinfomap.GetSet(or.IPAddress, make(chan time.Time, TimeThreshold))//如果不存在当前key则新增一个
	castchan := tchan.(chan time.Time)
	if len(castchan) >= TimeThreshold { //对于出口量的IP大于阀值的条件容易达到
		got := <-castchan
		if got.After(time.Now().Add(-time.Minute * time.Duration(TimeThreshold))) {
			return ResultMessage{ReturnCode: 1, Message: errmsg}
		}
	}
	value := time.Now()
	castchan <- value
	ipinfomap.Set(or.IPAddress, castchan)
```
 - 针对每日订单量的监测，使用以下代码获取统计表中的历史均值并缓存到redis中，下次直接使用redis获取的数据与当日订单数比对，使用redis中的incr方法累计当日订单量
```go
func GetAvgByKey(r redis.Conn, rediskey string, rawsql string, args ...interface{}) int {
	sumcount := 0
	avgcount, errg := redis.Int(r.Do("GET", rediskey))
	if errg == nil {
		log.Debug("get by key: " + string(rediskey))
		return avgcount
	}
	log.Error(errg.Error())
	results, err := orm.Query(rawsql, args...)
	if err != nil {
		log.Error(err.Error())
	}
	if len(results) > 10 { //相应的历史数据超过10天才作为参考
		for k, _ := range results {
			nowoc, _ := strconv.Atoi(string(results[k]["OrderCount"]))
			sumcount = sumcount + nowoc
		}
		avgcount = sumcount / len(results)
		log.Debug("get by sum: " + strconv.Itoa(avgcount))
	}
	r.Do("SETEX", rediskey, 86400, avgcount)//一天过期
	return avgcount
}
```
####存在的问题
- 不支持热部署，需要停止应用后替换应用，当前的方案是部署两台服务器，使用nginx作为前端代理，当需要更新应用是轮流停止应用上线

- 如果使用更长周期的刷单行为则无法监测到，例如持续的每天刷一部分订单数据，针对这种情况可能需要通过标示和历史数据分析来统计处有问题的数据的规律