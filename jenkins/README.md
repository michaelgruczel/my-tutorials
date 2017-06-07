# jenkins

All codesnippets which i will use here can be found in this folder.

## installation

jenkins is one of the existing ci systems.
You can get details here https://jenkins.io/index.html.
One way to run it is to download a war or an installer from the webpage and start it in a self-embedded way e.g. by

    java -jar jenkins.war
    
Jenkins is getting powerful by adding plugins (https://wiki.jenkins-ci.org/display/JENKINS/Plugins).
This can be done by hand in the UI, but that's not the way we want to do it. Doing things manually is the base for errors.
Fortunately there are simple ways to automate it (at least a lot of it).
One simple way is docker

Dockerfile:

    FROM jenkins
    MAINTAINER <your name here> "<your mail here>"
    
    USER jenkins
    COPY plugins.txt /usr/share/jenkins/plugins.txt
    RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt

plugins.txt:

    scm-api:latest
    git-client:latest
    git:latest
    greenballs:latest
    token-macro:1.11
    dockerhub:1.0
    disk-usage:0.25
    credentials:1.24

You can get the dependencies and versions from the plugins page.    
so, that's all you need to start a jenkins in a docker container with plugins

    docker build -t michael/jenkins .
    docker run -d -p 8080:8080 --name custom-jenkins michael/jenkins    
    
maybe you want to mount a shared folder with the docker container and additional ports e.g. like this
    
    docker run -d -p 8080:8080 -p 49187:49187 -p 50000:50000 -v /usr/share/jenkins-backup:/var/jenkins_home --name custom-jenkins michael/jenkins    
    
/var/jenkins_home is the folder within the container, /usr/share/jenkins-backup would be on you pc.
Maybe you have to set the userid of the user within the container to the same outside of the container.

    chown 1000:1000 /usr/share/jenkins-backup
    
and maybe you have to adapt se linux setting (su -c 'setenforce 0').
You should be able to see jenkins now on http://localhost:8080

## create jobs

You can create jenkins jobs by hand. But, that means you need a backup strategy for the jobs.

There are 3 options to do that:

* copy the job description somewhere (xml files can be retrieved from filesystems or by http call, or by jenkins-cli)
* install a plugin which commits every change to a scm
* generate the jobs automatically (infrastructure as code)

We will go for option 3

There is a gradle plugin which generates jenkins jobs by a dsl description which can be easiliy checked in.
See https://github.com/ghale/gradle-jenkins-plugin/wiki for more details, I will just give a short example

You need a local gradle wrapper (like in this folder of this repo) and a description for the job generation.
The test-job.gradle file would create a jenkins job named TEST-TUTORIAL-JOB at the local running jenkins.
I have skipped the setting up of users, so for this tutorial you have to do it by hand (sorry) and setup the user in the script.
The job would just write "hello world on the shell" and store the last 49 runs (7 days).
It should be self-explanatory.

test-job.gradle

    apply plugin: 'com.terrafolio.jenkins'

    buildscript {
        repositories {
            mavenLocal()
            mavenCentral()
            maven {
                url "https://plugins.gradle.org/m2/" // for gradle-plugins
            }
        }
        dependencies {
            classpath('com.terrafolio:gradle-jenkins-plugin:1.3.1')
        }
    }
    
    project.ext {
        projectName = 'TEST'
        useGradleStep = project.hasProperty('useGradleStep') && ['on', 'true'].contains(project.properties.useGradleStep)
        defaultReceipients = ''
        defaultSubject = ''
        defaultContent = ''
    }
    
    jenkins {
        servers {
            teamapps {
                url "http://localhost:8080/"
                username "<a user with change rights>"
                password "<a password>"
            }
        }

        defaultServer servers.teamapps

        jobs {
            metaJob {
                dsl {
                    name ("${projectName}-TUTORIAL-JOB")
                    description "funky test job for ${projectName}.<br>\nSee https://github.com/ghale/gradle-jenkins-plugin/wiki for DSL syntax."

                    steps {
                        shell('echo Hello World')
                    }
    
                    // discard old builds:
                    logRotator(7, 49) // daysToKeep, numBuildsToKeep
                }
            }
        }
    
    }

In order to execute it you just have to run and the job will be created (or updated if already existing)

    ./gradlew -b test-pipeline.gradle updateJenkinsItems

## pipelines

Let's say job 1 should trigger job 2 in case of a success, then we could define

    jobs {
        exampleJob {
            dsl {
                description "funky test job for ${projectName}.<br>\nSee https://github.com/ghale/gradle-jenkins-plugin/wiki for DSL syntax."

                //steps {
                //    powerShell('Write-Output "Hello World!"')
                //}

                // discard old builds:
                logRotator(7, 49) // daysToKeep, numBuildsToKeep
                publishers {
                    // use build-pipeline-plugin
                    // https://wiki.jenkins-ci.org/display/JENKINS/Build+Pipeline+Plugin
                    downstream('exampleJob2', 'SUCCESS')
                }
            }
        }

        exampleJob2 {
            dsl {
                description "funky test job for ${projectName}.<br>\nSee https://github.com/ghale/gradle-jenkins-plugin/wiki for DSL syntax."

                steps {
                    shell('ansible --version')
                }

                // discard old builds:
                logRotator(7, 49) // daysToKeep, numBuildsToKeep
            }
        }

        
    }

    ./gradlew -b test-job.gradle updateJenkinsItems

Using the jenkins build pipeline plugin (https://wiki.jenkins-ci.org/display/JENKINS/Build+Pipeline+Plugin), 
we can get a nice UI on top of it

     
So now you should be able to automize the most of your CI infrastructure by code.
