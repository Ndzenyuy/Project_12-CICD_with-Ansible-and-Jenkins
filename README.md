# Project 12: CICD Pipeline with Ansible and Jenkins

In this project, i deployed a pipeline integrating Ansible to Jenkins. Ansible is used to deploy our artifact to a Tomcat server. The artifact is downloaded from Nexus artifact repository and deployed to a Tomcat server, configured and installed through ansible. In case the deployment fails, ansible will then rollback to the previous functional artifact. The deployment is first done to staging servers, after necessary tests are carried out on this version, a human validation is required to push it to production. 

## Project architecture

![](architecture)

## Prereqs

- Continues Integration(Project 5)

## Steps

- Continues Integration and Webhook \
    This project builds on Project 5 where i implemented a Continues Integration. To proceed, we have to increase the EBS root volume from 8GiB to 15GiB. Make sure the Jenkins-server(EC2) is stopped, then select it -> Volumes -> volume_id -> action -> modify volume -> 15GiB -> modify(validate) and power on the instances: Jenkins-server, Nexus-server, Sonar-server\

    On to Github, edit the webhook and give it the new public ip and update webhook, make sure to test the continues integration pipeline to be sure it is working before proceeding.

- Prepare App server staging
    Launch an Ec2 instance:
    ```
    name: app01-staging
    OS: ubuntu 20
    create keypair: app-key.pem
    create sg: app-sg
        - rules: allow ssh from my ip
                 allow 8080 frim my ip
                 allow ssh from jenkins-sg
                
    ```

    Create a hosted zone in route 53
    ```
        name: vprofile.project
        type: Private hosted zone
        region: us-east-2 (same region where instances are run)
        vpc: choose default -> validate create
        Create record:
                name: app01stg
                record-type: A
                value: <private ip of app01-staging>
    ```

    Add ssh credentials to Jenkins: manage jenkins -> credentials -> global credentials -> add credentials
    ```
    kind: SSH username with private key
    scope: global
    ID: applogin
    description: applogin
    username: ubuntu
    private key: copy and paste the contents of app-key.pem
    save
    ```

- Ansible in Jenkins \
    First we SSH into Jenkins server, there we will install ansible by running the following commands

    ```
    sudo apt update
    sudo apt install software-properties-common -y
    sudo add-apt-repository --yes --update ppa:ansible/ansible
    sudo apt install ansible -y
    ```

    Back to Jenkins dashboard: manage jenkins -> plugins -> available: search for ansible then install
    
- Prepare source code \
    Clone the project source code
    ```
    git clone git@github.com:Ndzenyuy/Project_12-CICD_with-Ansible-and-Jenkins.git
    ```
    Copy the contents to a new project folder and initialise git with a remote repository

- Ansible Plabooks \
    In the project folder, we'll have a sub folder called templates, in it the following files should be present: \
    1. site.yml
    ```yml
    ---
        - import_playbook: tomcat_setup.yml
        - import_playbook: vpro-app-setup.yml 

        ####

    ```
    2. tomcat_setup.yml
    ```yml
    - name: Common tool setup on all the servers
    hosts: appsrvgrp
    become: yes
    vars:
        tom_url: https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.37/bin/apache-tomcat-8.5.37.tar.gz

    tasks:
        - name: Install JDK on Centos 6/7
        yum:
            name: java-1.8.0-openjdk.x86_64
            state: present
        when: ansible_distribution == 'CentOS'

        - name: Install JDK on Ubuntu 14/15/16/18
        apt:
            name: openjdk-8-jdk
            state: present
            update_cache: yes
        when: ansible_distribution == 'Ubuntu' 

        - name: Download Tomcat Tar Ball/Binaries
        get_url:
            url: "{{tom_url}}"
            dest: /tmp/tomcat-8.tar.gz

        - name: Add tomcat group
        group:
            name: tomcat
            state: present

        - name: Add tomcat user
        user:
            name: tomcat
            group: tomcat
            shell: /bin/nologin
            home: /usr/local/tomcat8

        - file:
            path: /tmp/tomcat8
            state: directory

        - name: Extract tomcat
        unarchive:
            src: /tmp/tomcat-8.tar.gz
            dest: /tmp/tomcat8/
            remote_src: yes
            list_files: yes
        register: unarchive_info

        - debug:
            msg: "{{unarchive_info.files[0].split('/')[0]}}"

        - name: Synchronize /tmp/tomcat8/tomcat_cont /usr/local/tomcat8.
        synchronize:
            src: "/tmp/tomcat8/{{unarchive_info.files[0].split('/')[0]}}/"
            dest: /usr/local/tomcat8/
        delegate_to: "{{ inventory_hostname }}"

        - name: Change ownership of /usr/local/tomcat8
        file:
            path: /usr/local/tomcat8
            owner: tomcat
            group: tomcat
            recurse: yes

        - name: Setup tomcat SVC file on Centos 7
        template:
            src: templates/epel7-svcfile.j2
            dest: /etc/systemd/system/tomcat.service
            mode: "a+x"
        when: ansible_distribution == 'CentOS' and ansible_distribution_major_version == '7'      

        - name: Setup tomcat SVC file on Centos 6
        template:
            src: templates/epel6-svcfile.j2
            dest: /etc/init.d/tomcat
            mode: "a+x"
        when: ansible_distribution == 'CentOS' and ansible_distribution_major_version == '6'

        - name: Setup tomcat SVC file on ubuntu 14/15
        template:
            src: templates/ubuntu14_15-svcfile.j2
            dest: /etc/init.d/tomcat
            mode: "a+x"
        when: ansible_distribution == 'Ubuntu' and ansible_distribution_major_version < '16'

        - name: Setup tomcat SVC file on ubuntu 16 and 18
        template:
            src: templates/ubuntu16-svcfile.j2
            dest: /etc/systemd/system/tomcat.service
            mode: "a+x"
        when: ansible_distribution == 'Ubuntu' and ansible_distribution_major_version >= '16'

        - name: Reload tomcat svc config in ubuntu 14/15
        command: update-rc.d tomcat defaults
        when: ansible_distribution == 'Ubuntu' and ansible_distribution_major_version < '16'

        - name: Reload tomcat svc config in Centos 6
        command: chkconfig --add tomcat
        when: ansible_distribution == 'CentOS' and ansible_distribution_major_version == '6'

        - name: just force systemd to reread configs (2.4 and above)
        systemd:
            daemon_reload: yes 
        when: ansible_distribution_major_version > '6' or ansible_distribution_major_version > '15'

        - name: Start & Enable TOmcat 8
        service:
            name: tomcat
            state: started
            enabled: yes

    ```

    3.  vpro-app-setup.yml
    ```yml    
        - name: Setup Tomcat8 & Deploy Artifact
        hosts: appsrvgrp
        become: yes
        vars:
            timestamp: "{{ansible_date_time.date}}_{{ansible_date_time.hour}}_{{ansible_date_time.minute}}"
        tasks:
            - name: Download latest VProfile.war from nexus
            get_url:
                url: "http://{{USER}}:{{PASS}}@{{nexusip}}:8081/repository/{{reponame}}/{{groupid}}/{{artifactid}}/{{build}}-{{time}}/{{vprofile_version}}"
                dest: "/tmp/vproapp-{{vprofile_version}}"
            register: wardeploy
            tags:
            - deploy

            - stat:
                path: /usr/local/tomcat8/webapps/ROOT
            register: artifact_stat
            tags:
            - deploy

            - name: Stop tomcat svc
            service:
                name: tomcat
                state: stopped
            tags:
            - deploy

            - name: Try Backup and Deploy
            block:
            - name: Archive ROOT dir with timestamp
                archive:
                path: /usr/local/tomcat8/webapps/ROOT
                dest: "/opt/ROOT_{{timestamp}}.tgz"
                when: artifact_stat.stat.exists
                register: archive_info
                tags:
                - deploy

            - name: copy ROOT dir with old_ROOT name
                shell: cp -r ROOT old_ROOT
                args:
                chdir: /usr/local/tomcat8/webapps/

            - name: Delete current artifact
                file:
                path: "{{item}}"
                state: absent
                when: archive_info.changed
                loop:
                - /usr/local/tomcat8/webapps/ROOT
                - /usr/local/tomcat8/webapps/ROOT.war
                tags:
                - deploy

            - name: Try deploy artifact else restore from previos old_ROOT
                block:
                - name: Deploy vprofile artifact
                copy:
                    src: "/tmp/vproapp-{{vprofile_version}}"
                    dest: /usr/local/tomcat8/webapps/ROOT.war
                    remote_src: yes
                register: deploy_info
                tags:
                    - deploy
                rescue:
                - shell: cp -r old_ROOT ROOT
                    args:
                    chdir: /usr/local/tomcat8/webapps/

            rescue:
            - name: Start tomcat svc
                service:
                name: tomcat
                state: started

            - name: Start tomcat svc
            service:
                name: tomcat
                state: started
            when: deploy_info.changed
            tags:
            - deploy

            - name: Wait until ROOT.war is extracted to ROOT directory
            wait_for:
                path: /usr/local/tomcat8/webapps/ROOT
            tags:
            - deploy

        #    - name: Deploy web configuration file
        #      template:
        #        src: templates/application.j2
        #        dest: /usr/local/tomcat8/webapps/ROOT/WEB-INF/classes/application.properties
        #        force: yes
        #      notify:
        #       - Restart Tomcat
        #      tags:
        #       - deploy

        handlers:
        - name: Restart Tomcat
            service:
            name: tomcat
            state: restarted

    ```

    4. ansible.cfg file
    ```
    [defaults]
    host_key_checking = False
    timeout = 30
    ```

    5. prod.inventory file
    ```
    [appsrvgrp]
    app01prod.vprofile.project
    ```
- Jenkinsfile 
```
        /*def COLOR_MAP [
            'SUCCESS': 'good',
            'FAILURE': 'danger',
        ]*/
        pipeline {
            agent any
            tools {
                maven "MAVEN3"
                jdk "OracleJDK8"
            }
            
            environment {
                SNAP_REPO = 'vprofile-snapshot'
                NEXUS_USER = 'admin'
                NEXUS_PASS = 'admin123'
                RELEASE_REPO = 'vprofile-release'
                CENTRAL_REPO = 'vpro-maven-central'
                NEXUSIP = '172.31.25.86'
                NEXUSPORT = '8081'
                NEXUS_GRP_REPO = 'vpro-maven-group'
                NEXUS_LOGIN = 'nexuslogin'
                SONARSERVER = 'sonarserver'
                SONARSCANNER = 'sonarscanner'
                NEXUSPASS = credentials('nexuspass')
                
            }

            stages {
                stage('Build'){
                    steps {
                        sh 'mvn -s settings.xml -DskipTests install'
                    }
                    post {
                        success {
                            echo "Now Archiving."
                            archiveArtifacts artifacts: '**/*.war'
                        }
                    }
                }

                stage('Test'){
                    steps {
                        sh 'mvn -s settings.xml test'
                    }

                }

                stage('Checkstyle Analysis') {
                    steps {
                        sh 'mvn -s settings.xml checkstyle:checkstyle'

                    }
                }

                stage('CODE ANALYSIS with SONARQUBE') {
                
                environment {
                    scannerHome = tool "${SONARSCANNER}"
                }

                steps {
                    
                    withSonarQubeEnv("${SONARSERVER}") {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"
                    }

                    

                    /*timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                    }*/
                }
                }  

                stage("UploadArtifact"){
                    steps{
                        nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                        groupId: 'QA',
                        version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                        repository: "${RELEASE_REPO}",
                        credentialsId: "${NEXUS_LOGIN}",
                        artifacts: [
                            [artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war']
                        ]
                        )
                    }
                }  

                stage('Ansible Deploy to staging'){
                    steps {
                        ansiblePlaybook([
                        inventory   : 'ansible/stage.inventory',
                        playbook    : 'ansible/site.yml',
                        installation: 'ansible',
                        colorized   : true,
                        credentialsId: 'applogin',
                        disableHostKeyChecking: true,
                        extraVars   : [
                            USER: "admin",
                            PASS: "${NEXUSPASS}",
                            nexusip: "172.31.25.86",
                            reponame: "vprofile-release",
                            groupid: "QA",
                            time: "${env.BUILD_TIMESTAMP}",
                            build: "${env.BUILD_ID}",
                            artifactid: "vproapp",
                            vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war"
                        ]
                    ])
                    }
                }

                        
            

                
            }

            /*post {
                always {
                    echo 'Slack Notifications'
                    slackSend channel: '#jenkinscicd',
                        colo: COLOR_MAP[currentBuild.currentResult],
                        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}"
                }
            }*/
        }
```

- App server for staging \
    On Jenkins dashboard, create a new pipeline item: cicd-jenkins-ansible-staging
    ```
    Github hook trigger for GITScm polling = true
    pipeline: Pipeline script from scm
        SCM: Git
        repository URL: <your ssh repository url>
        credentials: git(githublogin) [created in project 5]
        branch: */main
    ```
    Save and Build project. 
    ![](build success)
    ![](App deployed to stage)

- App server for prod \
    Launch an Ec2 instance:
    ```
    name: app01-Prod
    OS: ubuntu 20
    create keypair: app-prod-key.pem
    create sg: app-sg
        - rules: allow ssh from my ip
                 allow 8080 frim my ip
                 allow ssh from jenkins-sg
                
    ```
- Jenkinsfile for prod
 ```
    /*def COLOR_MAP [
        'SUCCESS': 'good',
        'FAILURE': 'danger',
    ]*/
    pipeline {
        agent any
        
        environment {
            NEXUSPASS = credentials('nexuspass')
            
        }

        stages {
            stage('Setup parameters') {
                steps {
                    script{
                        properties([
                            parameters([
                                string(
                                    defaultValue: '',
                                    name: 'BUILD',
                                ),
                                string(
                                    defaultValue: '',
                                    name: 'TIME'
                                )
                            ])
                        ])
                    }
                }
            }
            stage('Ansible Deploy to prod'){
                steps {
                    ansiblePlaybook([
                    inventory   : 'ansible/prod.inventory',
                    playbook    : 'ansible/site.yml',
                    installation: 'ansible',
                    colorized   : true,
                    credentialsId: 'applogin-prod',
                    disableHostKeyChecking: true,
                    extraVars   : [
                        USER: "admin",
                        PASS: "${NEXUSPASS}",
                        nexusip: "172.31.25.86",
                        reponame: "vprofile-release",
                        groupid: "QA",
                        time: "${env.TIME}",
                        build: "${env.BUILD}",
                        artifactid: "vproapp",
                        vprofile_version: "vproapp-${env.BUILD}-${env.TIME}.war"
                    ]
                ])
                }
            }

                    
        

            
        }

        /*post {
            always {
                echo 'Slack Notifications'
                slackSend channel: '#jenkinscicd',
                    colo: COLOR_MAP[currentBuild.currentResult],
                    message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}"
            }
        }*/
    }

 ```
 
- Create another record in vprofile.project hosted zone in route 53
    ```
        name: app01prod
        record-type: A
        value: <private ip of app01-prod>
    ```

-    Add ssh credentials to Jenkins: manage jenkins -> credentials -> global credentials -> add credentials

    ```
    kind: SSH username with private key
    scope: global
    ID: applogin-prod
    description: applogin-prod
    username: ubuntu
    private key: copy and paste the contents of app-prod-key.pem(keypair used to loging to prod server)
    -> save
    ```
- Jenkinsfile for prod
    Create a new git branch from our project folder in vscode
    ```
    git checkout -b cicd-jenkans-prod
    ```
    Rename stage.inventory to prod.inventory, in the code change ```app01stg.vprofile.project``` to ```app01prd.vprofile.project```
    In the Jenkins file,
    ```
        pipeline {
            agent any
            
            environment {
                NEXUSPASS = credentials('nexuspass')
                
            }

            stages {
                stage('Setup parameters') {
                    steps {
                        script{
                            properties([
                                parameters([
                                    string(
                                        defaultValue: '',
                                        name: 'BUILD',
                                    ),
                                    string(
                                        defaultValue: '',
                                        name: 'TIME'
                                    )
                                ])
                            ])
                        }
                    }
                }
                stage('Ansible Deploy to prod'){
                    steps {
                        ansiblePlaybook([
                        inventory   : 'ansible/prod.inventory',
                        playbook    : 'ansible/site.yml',
                        installation: 'ansible',
                        colorized   : true,
                        credentialsId: 'applogin-prod',
                        disableHostKeyChecking: true,
                        extraVars   : [
                            USER: "admin",
                            PASS: "${NEXUSPASS}",
                            nexusip: "172.31.25.86",
                            reponame: "vprofile-release",
                            groupid: "QA",
                            time: "${env.TIME}",
                            build: "${env.BUILD}",
                            artifactid: "vproapp",
                            vprofile_version: "vproapp-${env.BUILD}-${env.TIME}.war"
                        ]
                    ])
                    }
                }

                        
            

                
            }

            /*post {
                always {
                    echo 'Slack Notifications'
                    slackSend channel: '#jenkinscicd',
                        colo: COLOR_MAP[currentBuild.currentResult],
                        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}"
                }
            }*/
        }
    ```
    Save and run pipeline on Jenkins
    ![](pipeline prod)
    For the parameters, we have to pass them before building, so the artifact which we require to build is selected, the prod pipeline awaits the build number and the timestamp. We set these parameters and the pipeline builds it after the QA team gives a go ahead

- Complete CICD flow \
    The prod pipeline needs parameters to run, thus we have to remove the Git Polling webhook flag that was set in the pipeline configurations: Configure pipeline -> GitHub hook trigger for GITScm polling = false -> save
    
