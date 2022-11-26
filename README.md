# docker_jupyterlab_torch_June
docker deploy app for jupyter-lab with torch1.10, cuda 11.3 
os: linux arm64(tested)

## From docker file building to jupyter lab setting
1. download docker_deploy_app.zip from release
2. unzip docker_deploy_app.zip
> you will have 3 files. Dockerfile, init.sh, docker_bash.sh
- Dockerfile: to build docker
- init.sh: co-work with Dockerfile, list module nedd to be installed by apt-get(docker RUN seems like without ability to sudo apt-get)
- docker_bash.sh: UI to build, run, exec, stop your docker image.
3. run `bash docker_bash.sh your-image-name`
> four options is appeared:
(a) build       (b) run                                                                                                 
(c) exec        (d) stop
> You have to select (a) to build the image first, then (b) to run it. After all, u can (c) exec it. If needed, select (d) to stop it. 
> you might need to remove existed image, cmd is `sudo rm -f your-image-name`
> If u select (b), an additional input is needed to specified port for docker container and phisical machine. This port(default:8899) will be used to connect jupyter lab. (https://127.0.0.1:8899) 
  > if container is on remote server, https://remote-server-ip:8899
4. jupyter lab will auto start if everything is ok. but if stop.
```
bash docker_bash.sh your-image-name
# select (c) exec, you will auto turn into docker container edit session.
nohup jupyter lab & # run jupyter lab on background
```

### enjoy!


## intorduction
###### build
![image](https://user-images.githubusercontent.com/32012425/204074320-bbc707fe-d426-465e-b15f-23d713e39da0.png)
> 記得要加上image-name
###### 結果
![image](https://user-images.githubusercontent.com/32012425/204074545-1d238670-3197-4bb0-a1e6-6257899acb77.png)
###### docekr run
![image](https://user-images.githubusercontent.com/32012425/204074556-30011adc-fd16-4d43-a385-817474f063bd.png)
###### 執行
![image](https://user-images.githubusercontent.com/32012425/204074578-f222ff69-fdb8-4fe7-9d66-4dc1bb168f2d.png)


### Dockerfile 
```
# linux arm64, Ubuntu 18.04.5 LTS"
FROM pytorch/pytorch:1.10.0-cuda11.3-cudnn8-devel #

MAINTAINER june "bc165870081@gmail.com" #你好

RUN rm /bin/sh && ln -s /bin/bash /bin/sh # 把sh編譯器砍掉，指定使用bash
RUN echo yes | pip install --no-cache-dir -q jupyterlab # install jupyter lab


RUN echo yes | jupyter notebook --generate-config && \ # 產生juputer 設定文件 jupyter_notebook_config.py
defaultPath="/root/.jupyter/jupyter_notebook_config.py" && \ #指定設定文件位置
sed -i -e '82c c.NotebookApp.allow_remote_access = True' "$defaultPath" && \ # 允許遠端
sed -i -e '204c c.NotebookApp.ip = "*"' "$defaultPath" && \ # 允許遠端
sed -i -e '267c c.NotebookApp.open_browser = False' "$defaultPath" && \  # 不打開瀏覽器，ssh用
sed -i -e '287c c.NotebookApp.port = 8899' "$defaultPath" && \  # 指定到8899的port
sed -i -e '178c c.NotebookApp.allow_root = True' "$defaultPath" && \ # root權限，docker container必須
sed -i -e '469c c.NotebookApp.password = ""' "$defaultPath" && \ # (重要)不需要密碼，有潛在的安全疑慮
sed -i -e '340c c.NotebookApp.token = ""' "$defaultPath" && \ # (重要)不需要密碼，有潛在的安全疑慮
sed -i -e '284c c.NotebookApp.password_required = False' "$defaultPath"  # (重要)不需要密碼，有潛在的安全疑慮
# allow_root is needed

ADD init.sh . # 把要用apt-get安裝的東西寫成bash載入
RUN bash init.sh # apt-get install 各種模組

ENV SHELL=/bin/bash
```

### docker_bash.sh
```
#!/bin/bash
name=$1

#code to ignore case restrictions
shopt -s nocasematch

#case statement
echo -e "(a) build\t(b) run\n(c) exec\t(d) stop"
echo "Please enter the task to do"
read task
case $task in
	a)
		sudo docker build --rm -t $name . --no-cache
		echo " Build image {$name}"
		;;
	b)
		echo "set port for jupyter (default: 8899"		
		read port
		dst=$HOME/junetest/docker_mount/$name
		mkdir $dst
        args=(--gpus all # 連結gpu
                --restart unless-stopped # 自動重啟
		       	--name $name # 容器的命名
                -p $port:$port  # 通訊埠小聯通
                -v $dst:/workspace # 掛載，讓container和實體機器有可以共同存取的空間
		       	$name:latest # 使用的image名稱(剛才build的
		       	/bin/bash -c "jupyter lab"   # 當啟動，順便用bash把jupyter lab開起來，有其他工作想執行也可以放這裡
             )
        sudo docker run -idt "${args[@]}"

		echo " Run docker image $name, port $port, mount at $dst"
		;;
	c)
		sudo docker exec -u root -ti $name bash
		echo " Exec docker image"
		;;
	d)
		echo "stop docker image {$name}"
		sudo docker stop $name
		;;
	*)
		echo "Couldn’t find availbale task"
esac
```
