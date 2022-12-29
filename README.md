# java-package

# Dockerfile
FROM abrtech/alpine-oraclejdk8
MAINTAINER supernode.com
LABEL commitid=
ENV TZ=Asia/Shanghai
ADD ./run.sh /
RUN ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai'  > /etc/timezone && \
    chmod 777 /run.sh
COPY ./target/ai-mark.jar app.jar
EXPOSE 8010/tcp
ENTRYPOINT [ "sh", "-c", "/run.sh" ]


#run.sh
exec java  -jar -Dspring.profiles.active=prod /app.jar



#jenkins
flag=`date '+%Y%m%d%H%M'`
comitid=`git log |grep commit |awk '{print $2}' | head -n 1`
cd ${WORKSPACE}/
#echo "3fd3c265ce42ecf8e10e8996857a6536ed2b5e98">old_commit_commitid.txt
find ./ -name run.sh|xargs sed -i s#prod#dev#g
latest_commit_commitid=`git log|grep commit|head -2|sed -n '1p'|awk '{print $2}'`
old_commit_commitid=`cat old_commit_commitid.txt`
#old_commit_commitid="9cb93c1d2bb7bf24632b86b1a6fd95d018777b5d"
echo "%%%%%%%%%%%%%%%% latest_commit_commitid:$latest_commit_commitid "
echo "%%%%%%%%%%%%%%%% old_commit_commitid:$old_commit_commitid "
echo "${latest_commit_commitid}" > /tmp/old_commit_commitid.txt
echo "${latest_commit_commitid}" > old_commit_commitid.txt


deploy_service(){

    echo "*********************************************开始部署 ${service_name} 服务**********************************************"
    docker -H tcp://192.168.1.120:4243 service update --force --image ${image_name} ${swarm_service_name}
    echo "*********************************************部署 ${service_name} 完成**********************************************"
    
}

build_images(){
    
    sed -i "s/LABEL commitid=/LABEL commitid=${comitid}/" Dockerfile

    echo "*********************************************开始制作 ${service_name} 服务docker镜像**********************************************"
    docker -H tcp://192.168.1.124:4243 build -t ${image_name} .
    #cp -fr ./target/*.jar /var/jenkins_home/run_jar/
    echo "*********************************************推送镜像：${image_name}**********************************************"
    docker -H tcp://192.168.1.124:4243 push ${image_name}
    echo "*********************************************${service_name} 服务docker镜像制作完成**********************************************"


}

build(){
    echo "*********************************************开始 mvn 构建，构建 全部 服务**********************************************"
    cd ${WORKSPACE}/
    /usr/apache-maven-3.6.3/bin/mvn clean install -Dmaven.test.skip=true -Dmaven.repo.local=/var/jenkins_home/.m2/repository
    echo "********************************************* base 服务构建完成 **********************************************"
}

build_and_deploy(){
    echo "需要部署的服务："
    diff_proj_name=`git diff --name-status ${latest_commit_commitid} ${old_commit_commitid} --stat|grep ai|grep -v '.idea'|grep -v 'ai-platform.iml'|awk  '{print $2}'|awk -F/  '{print $1}'| uniq `
    echo ${diff_proj_name} | while read line
    do
        echo "********************************************* 需要编译的服务 ${line}"

        for service_name in ${line}
        do
            echo "********************************************* 正在编译 ${service_name}"
            cd ${WORKSPACE}/${service_name}
            image_name=supernode:5000/${service_name}-oil-field-dev:${flag}
            if [ ! -f Dockerfile ];then
                continue
            else
                build_images
                swarm_service_name=oil-field-b-dev_oil-field-${service_name}-dev
                deploy_service
            fi
        done 
        
    done

}

build_all_deploy(){
    for service_name in ${diff_proj_name}
    do
        echo "********************************************* 正在编译 ${service_name}"
        cd ${WORKSPACE}/${service_name}
        image_name=supernode:5000/${service_name}-oil-field-dev:${flag}
        if [ ! -f Dockerfile ];then
            continue
        else
            build_images
            swarm_service_name=oil-field-b-dev_oil-field-${service_name}-dev
            deploy_service
        fi
    done

}

is_all(){
    build
    if [ ${all} = 1 ];then
    	echo "********************************************* 全部构建 *********************************************"
        
        diff_proj_name="patrol-flowable patrol-gateway patrol-system patrol-business patrol-infrastructure patrol-standard patrol-message patrol-census"
        #diff_proj_name="patrol-flowable"
        build_all_deploy
    else
    	echo "********************************************* 部分构建 *********************************************"
        main
    fi

}

main(){
build_and_deploy
}

is_all
