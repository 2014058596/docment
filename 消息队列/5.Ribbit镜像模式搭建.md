```sh


拉取镜像
docker pull rabbitmq:3.7.3-management


创建文件
mkdir rabbitmqcluster
cd rabbitmqcluster/
mkdir rabbitmq01 rabbitmq02 rabbitmq03

增加host目录   rabbitmq01  rabbitmq02

//启动容器
docker run -d --hostname rabbitmq01 --name rabbitmqCluster01 -v /home/soft/rabbitmqcluster/rabbitmq01:/var/lib/rabbitmq -p 15672:15672 -p 5672:5672 -e RABBITMQ_ERLANG_COOKIE='rabbitmqCookie' rabbitmq:3.7.3-management


docker run -d --hostname rabbitmq02 --name rabbitmqCluster02 -v /home/soft/rabbitmqcluster/rabbitmq02:/var/lib/rabbitmq -p 15673:15672 -p 5673:5672 -e RABBITMQ_ERLANG_COOKIE='rabbitmqCookie'  --link rabbitmqCluster01:rabbitmq01 rabbitmq:3.7.3-management

docker run -d --hostname rabbitmq03 --name rabbitmqCluster03 -v /home/soft/rabbitmqcluster/rabbitmq03:/var/lib/rabbitmq -p 15674:15672 -p 5674:5672 -e RABBITMQ_ERLANG_COOKIE='rabbitmqCookie'  --link rabbitmqCluster01:rabbitmq01 --link rabbitmqCluster02:rabbitmq02 rabbitmq:3.7.3-management


启动成功后
重复服务
docker exec -it rabbitmqCluster01 bash
rabbitmqctl stop_app 
rabbitmqctl reset 
rabbitmqctl start_app 
exit

docker exec -it rabbitmqCluster02 bash
rabbitmqctl stop_app 
rabbitmqctl reset 
rabbitmqctl join_cluster --ram rabbit@rabbitmq01 
rabbitmqctl start_app 
exit


docker exec -it rabbitmqCluster03 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster  rabbit@rabbitmq01
rabbitmqctl start_app
exit




```

