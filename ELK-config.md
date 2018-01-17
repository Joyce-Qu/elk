ELK 6.1.0 config（for mac）

- elasticsearch
    - 下载
        `curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.1.0.tar.gz`
        `tar -xvf elasticsearch-6.1.0.tar`
    - 配置
    1. 创建ELK用户组(系统设置添加用户组)
    2. 创建/Users/elasticsearch目录
    3. 创建elasticsearch用户，设置用户目录，密码（执行命令用sudo）
        ```
        sudo dscl . -create /Users/elasticsearch
        sudo dscl . -create /Users/elasticsearch UserShell /bin/bash
        sudo dscl . -create /Users/elasticsearch RealName elasticsearch
        sudo dscl . -create /Users/elasticsearch UniqueID “1010”
        sudo dscl . -create /Users/elasticsearch PrimaryGroupID 80
        sudo dscl . -create /Users/elasticsearch NFSHomeDirectory /Users/elasticsearch
        sudo dscl . -passwd /Users/elasticsearch Pass1234
        sudo cp -R /System/Library/User\ Template/English.lproj /Users/elasticsearch/
        sudo chown -R elasticsearch:staff /Users/elasticsearch/
        ```
    4. elasticsearch加入ELK用户组
        ```
        sudo dscl . -append /Groups/ELK GroupMembership elasticsearch
        ```
    5. 登录elasticsearch用户
        ```
        su - elasticsearch
        ```
    6. 把elasticsearch/logstash解压到/User/elasticsearch目录下
    7. 安装x-pack
        ```
        elasticsearch-6.1.0/bin/elasticsearch-plugin install --batch x-pack
        ```
    8. 配置elasticsearch.yml(0.0.0.0意思匹配所有本机所有ip)
        - cluster.name: es
        - node.name:<主机名>
        - network.host: 0.0.0.0
    
    9. 启动elasticsearch
        ```
        bin/elasticsearch
        ```

    10. 改x-pack默认密码(分别改为kibana/elastic/logstash)
        ```
        elasticsearch-6.1.0/bin/x-pack/setup-passwords interactive
        ```
        用改过密码可以登录http://127.0.0.1:9200/
    11. stop elasticsearch
        - 
        ```
            ps | grep Elasticsearch
            kill -SIGTERM 13343
        ```



- logstash
    <!-- - launch logstash
        - `bin/logstash -f logstash-simple.conf`
    - send message
        - `curl -d '{"word":6}' "http://localhost:20005"` -->

    1. 启动logstash(本例logstash等待用户输入)
        ```
        bin/logstash -f jenkins-pipeline.conf
        ```
        配置jenkins-pipeline.conf(配置用户名密码elastic，输出显示并输出到elasticsearch,输出到9200端口)
        ```
        input {
            udp {
                host        => "0.0.0.0"
                type        => "after_scenario"
                port        => "20003"
                codec       => "json"
                tags        => [  ]
                buffer_size => 30720
            }
        }

        output {
            if [type] == "after_scenario" {
                elasticsearch {
                    hosts => ["127.0.0.1:9200"]
                    index => "after_scenario-%{+YYYY.MM.dd}"
                    user => "elastic"
                    password => "elastic"
                }
                stdout { codec => rubydebug}
            }
        }
        ```
    2. 改TA配置，跑case得到输入数据
    3. 查elasticsearch有没有收到数据（可以看到after_scenario-xxxd的index）
        ```
        curl -XGET 'localhost:9200/_cat/indices?v&pretty'  --user elastic:elastic
        ```
    4. 查找index的具体内容
        ```
        curl -XGET 'localhost:9200/after_scenario-2018.01.16/_search?pretty' -H 'Content-Type: application/json' -d'
        {
        "query": { "match_all": {} },
        "_source": ["tags", "login_name"]
        }
        ' --user elastic:elastic
        ```
    5. kibana搜索数据
        - 创建index-pattern
        - 根据规则搜索显示数据




- kibana
    1. 下载kibana
        ```
        curl -O https://artifacts.elastic.co/downloads/kibana/kibana-6.1.0-darwin-x86_64.tar.gz
        shasum kibana-6.1.0-darwin-x86_64.tar.gz 
        tar -xzf kibana-6.1.0-darwin-x86_64.tar.gz
        cd kibana-6.1.0-darwin-x86_64/ 
        ```
    2. 安装x-pack(超级慢)
        ```
        bin/kibana-plugin install x-pack
        ```
    3. 配置kibana.yml
        ```
        server.host: "0.0.0.0"
        elasticsearch.url: "http://localhost:9200"
        kibana.index: ".kibana"
        elasticsearch.username: "elastic"
        elasticsearch.password: "elastic"
        ```
    4. 启动kibana
    5. 登录kibana(注意用elastic用户而不是kibana，否则会看不到index)



- 遇到的问题
    - 最好把所有端口都改掉，不用默认端口以防引起冲突
    - 查看logstash状态
        - `curl -XGET 'localhost:9200/_cat/health?v&pretty'`
            遇到错误认证失败
        ```
        {
            "error" : 
            {
                "root_cause" : [
                {
                    "type" : "security_exception",
                    "reason" : "missing authentication token for REST request [/_cat/health?v&pretty]",
                    "header" : 
                    {
                        "WWW-Authenticate" : "Basic realm=\"security\" charset=\"UTF-8\""
                    }
                }
                ],
                "type" : "security_exception",
                "reason" : "missing authentication token for REST request [/_cat/health?v&pretty]",
                "header" : 
                {
                    "WWW-Authenticate" : "Basic realm=\"security\" charset=\"UTF-8\""
                }
            },
            "status" : 401
        }
        ``` 
        解决办法是添加上账号密码就好了
        ```
        curl --user elastic:elastic -XGET 'localhost:9200/_cat/health?v&pretty' 
        ```
    - kibana登录后看不到index
        登录用户错误，应该用elastic用户登录，而不是kibana(但是why？)


- *Jenkins slave ELK 配置*
    - elasticsearch.yml
        ```
            cluster.name: wme-monitor
            node.name: MACMINI-HF-04
            path.data: /Users/jenkins/workspace/tools/elk/elasticsearch-6.1.2/data
            path.logs: /Users/jenkins/workspace/tools/elk/elasticsearch-6.1.2/logs
            network.host: 0.0.0.0
            http.port: 9203
        ```
    - jenkins-pipeline.conf
        ```
        input {
            udp {
                host        => "0.0.0.0"
                type        => "after_scenario"
                port        => "20002"
                codec       => "json"
                tags        => [  ]
                buffer_size => 30720
            }
        }

        output {
            if [type] == "after_scenario" {
                elasticsearch {
                    hosts => ["127.0.0.1:9203"]
                    index => "after_scenario-%{+YYYY.MM.dd}"
                    user => "elastic"
                    password => "elastic"
                }
                stdout { codec => rubydebug}
            }
        }
        ```
    - logstash.yml
    - kibana.yml
        ```
        server.port: 5603
        server.host: "0.0.0.0"
        elasticsearch.url: "http://localhost:9203"
        kibana.index: ".kibana"
        elasticsearch.username: "elastic"
        elasticsearch.password: "elastic"
        ```
    - 开机自启动ELK脚本start_elk.sh(该脚本放在elk目录同级，elk目录内部是kibana/elasticsearch/logstash)
        ```
        #!/bin/bash

        SCRIPT_PATH=`dirname $0`
        print $SCRIPT_PATH
        echo "start elasticsearch"
        (cd $SCRIPT_PATH/elk/elasticsearch-6.1.2/bin/elasticsearch  &)
        echo "start kibana"
        (cd $SCRIPT_PATH/elk/kibana-6.1.2-darwin-x86_64/bin/kibana &)
        echo "start logstash"
        (cd $SCRIPT_PATH/elk/logstash-6.1.2/bin/logstash -f $SCRIPT_PATH/elk/logstash-6.1.2/jenkins-pipeline.conf &)
        echo "start status"
        (cd $SCRIPT_PATH; ./status.sh)
        ```
    - 查看ELK状态status.sh
        ```
        #!/bin/bash
        SCRIPT_PATH=`dirname $0`
        ps -ef | grep --color=always "elasticsearch[^ ]*\|logstash[^ ]*\|node" | grep -v "grep"
        ```

- 参考链接
    - <https://www.elastic.co/start>