podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: gradle
        image: gradle:6.3-jdk14
        command:
        - sleep
        args:
        - 99d
        volumeMounts:
        - name: shared-storage
          mountPath: /mnt
      - name: kaniko
        image: gcr.io/kaniko-project/executor:debug
        command:
        - sleep
        args:
        - 9999999
        volumeMounts:
        - name: shared-storage
          mountPath: /mnt
        - name: kaniko-secret
          mountPath: /kaniko/.docker
      restartPolicy: Never
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: jenkins-pv-claim
      - name: kaniko-secret
        secret:
            secretName: dockercred
            items:
            - key: .dockerconfigjson
              path: config.json
''') {
  node(POD_LABEL) {
    stage('Run pipeline against a gradle project') {
        git branch: 'main', url: 'https://github.com/vijayvad/week6'
        container('gradle') {
            try {
              if(env.BRANCH_NAME != "playground") {
                stage("Unit test") {
                  sh '''
                  ./gradlew test
                  '''
                }
                if (env.BRANCH_NAME != "feature") {
                stage("Code coverage!") {
                        sh '''
                            pwd
                            ./gradlew jacocoTestReport
                            ./gradlew jacocoTestCoverageVerification
                        '''
                    publishHTML (target: [
                        reportDir: 'build/reports/jacoco/test/html',
                        reportFiles: 'index.html',
                        reportName: "JaCoCo Report"
                    ])
                    }
                  }
                stage("Static code analysis!") {
                        sh '''
                        pwd
                        ./gradlew checkstyleMain
                        '''
                        publishHTML (target: [
                            reportDir: 'build/reports/checkstyle/',
                            reportFiles: 'main.html',
                            reportName: "Checkstyle Report"
                        ])
                    }
              }
                stage('Build a gradle project!') {
                    sh '''
                    pwd
                    chmod +x gradlew
                    ./gradlew build
                    mv ./build/libs/calculator-0.0.1-SNAPSHOT.jar /mnt
                    '''
                    }
            } catch (Exception E) {
                echo 'Failure detected'
            }
        }
    }
    if (env.BRANCH_NAME != "playground") {
      stage('Build Java Image') {
        container('kaniko') {
          stage('Build a container') {
            if (env.BRANCH_NAME == "main") {
              image_version = ":1.0"
            }
            if (env.BRANCH_NAME == "feature") {
              image_version = "-feature:0.1"
            }
           sh '''
                echo 'FROM openjdk:8-jre' > Dockerfile
                echo 'COPY ./calculator-0.0.1-SNAPSHOT.jar app.jar' >> Dockerfile
                echo 'ENTRYPOINT ["java", "-jar", "app.jar"]' >> Dockerfile
                mv /mnt/calculator-0.0.1-SNAPSHOT.jar .
                /kaniko/executor --context `pwd` --destination vijayvad/calculator${image_version}
                '''
          }
        }
      }
    }
  }
}
