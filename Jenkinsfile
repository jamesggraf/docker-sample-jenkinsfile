node {
    // https://registry.hub.docker.com/_/maven/
    def maven32 = docker.image('maven:3.2-jdk-7-onbuild');
    
    stage 'Mirror'
    // First make sure the slave has this image.
    // (If you could set your registry below to mirror Docker Hub,
    // this would be unnecessary as maven32.inside would pull the image.)
    maven32.pull()
    // We are pushing to a private secure docker registry in this demo.
    // 'docker-registry-login' is the username/password credentials ID
    // as defined in Jenkins Credentials.
    // This is used to authenticate the docker client to the registry.
    docker.withRegistry('https://docker.example.com/', 'docker-registry-login') {
    
        stage 'Build'
    
        // spin up a Maven container to build the petclinic app from source
        // (we are only using -v here to share the Maven local repository across demo runs;
        // otherwise would set localRepository=${pwd}/m2repo)
        maven32.inside('-v /m2repo:/m2repo') {
            // Check out the source code.
            git 'https://github.com/tfennelly/spring-petclinic.git'
    
            // Set up a shared Maven repo so we don't need to download all dependencies on every build.
            writeFile file: 'settings.xml',
                      text: '<settings><localRepository>/m2repo</localRepository></settings>'
            // Build with Maven settings.xml file that specs the local Maven repo.
            sh 'mvn -B -s settings.xml clean install -DskipTests'
    
            // The app .war and Dockerfile are now available in the workspace. See below.
        }
        
        stage 'Bake Docker image'
        // use the spring-petclinic Dockerfile (see above 'maven32.inside()' block)
        // to build container that can run the app. The Dockerfile is in the cwd of the active workspace
        // (see above maven32.inside() block), so we pass '.' as the build PATH param. The Dockerfile
        // (see https://github.com/tfennelly/spring-petclinic/blob/master/Dockerfile) expects the petclinic.war
        // file to be in the 'target' dir of the workspace, which will be the case.
        def pcImg = docker.build("examplecorp/spring-petclinic:${env.BUILD_TAG}", '.')
    
        // Let's tag and push the newly built image. Will tag using the image name provided
        // in the 'docker.build' call above (which included the build number on the tag).
        pcImg.push();
    
        stage 'Test Image'
        // Run the petclinic app in its own docker container
        pcImg.withRun {petclinic ->
            // Spin up a maven test container, linking it to the petclinic app container allowing
            // the maven tests to fire HTTP requests between the containers.
            maven32.inside("-v /m2repo:/m2repo --link=${petclinic.id}:petclinic") {
                git 'https://github.com/tfennelly/spring-petclinic-tests.git'
    
                writeFile file: 'settings.xml',
                          text: '<settings><localRepository>/m2repo</localRepository></settings>'
                sh 'mvn -B -s settings.xml clean package'
            }
        }
    
        stage name: 'Promote Image', concurrency: 1
        // All the tests passed. We can now retag and push the 'latest' image
        pcImg.push('latest');    
    }
}
