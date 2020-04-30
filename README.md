We are going to create an app that modifies a 'feature' branch, then it test it on Travis CI and if the test is passed it merge to 'master' branch and then upload to a AWS host in Beanstalk

![Image description](https://github.com/jorgeautomation/docker-react/blob/master/image-structure.png)

So we are going to run three different NPM commands, one for dev environment, one for the tests and another one for the build (app in production) which will use a light web server: NGINX

![Image description](https://github.com/jorgeautomation/docker-react/blob/master/image-npm.png)

PRE REQUISITES
=============

- Have nodeJS installed
- Have docker installed
- Github account
- Travis CI
- AWS account

1 CREATE THE DEV APP ENVIRONMENT
--------------------------------

NOTE: if using the repository in github, just download and run 'npm install'

1. run 'npx create-react-app frontend' in your command line to generate the app on the fly to get the most current libraries and avoid many dependency conflicts. This will create a project named 'frontend'

    - Note: In MAC there is an error when running 'npm run test' so first run this command 'brew install watchman'

2. Go to 'frontend' folder in cmd and run the following commands to understand how they work
    - 'npm run test' to check the test associated with the project
    - 'npm run build' to compact eveything into a single file, it also creates a 'build' folder
    - 'npm run start' to start our development server, it will automatically start app on localhost:3000

2 CREATE DOCKER CONTAINER FOR APP IN DEV
--------------------------------------

1. Inside 'frontend' create a Dockerfile.dev this will be the container for dev, where we install dependencies, copy all the files, and start app.

3. In the folder run: 'docker build -f Dockerfile.dev .' notice that we added -f Dockerfile.dev as .dev is not an extension default for docker build so we have to specify that

4. now we can remove the node_modules inside our working directory as they are now on the image, so the follwing will not be a lot of megas now as we don't have now the 'node_modules': 'Sending build context to Docker daemon  1.504MB'

5. **Run the container**, we have to add -it because of react app library: 'docker run -it -p 3000:3000 <imageid>'

3 CREATE DOCKER VOLUME FOR THE DEV CONTAINER TO AVOID REBUILDING EACH TIME THERE IS A CHANGE
----------------------------------------------------------------------------------------

1. Add this to your command: *docker run -it -p 3000:3000 -v /$(pwd):/app <imageid>*
    - Note: In Windows, you have to run it on gitbash as per $(pwd) is a linux command to get the current directory, also the '/' before $ is just for Windows
    - Note: In Windows you may get this error: npm ERR! enoent ENOENT: no such file or directory, open '/app/package.json, go to settings in docker for windows, go to shared drives -> reset credentials, then check drive C and restart docker.


2. A code NOENT for node_modules will appear. The issue is because we don't have node_modules folder in our local as we delete it. For that we have to use 'v /app/node_modules' so it would be a placeholder for the folder that is inside the container (remember we alredy have that folder in our image), so do not map it agains an external folder.
    - So run: *docker run -it -p 3000:3000 -v /app/node_modules -v $(pwd):/app <imageid>*

3. Now you can go to the app.js in the local folder, make a change and it will be reflected in the page without having to re buid the image.


4 CREATE DOCKER COMPOSE
-------------------------

1. We have an issue now, the thing is that the command to run the container now is pretty big. So we can use a docker compose file to include the volumes and the ports to make the command shorter, so it will be
    - we have to use 'dockerfile' inside we as the file has a differente extension (that is: dev)
    - docker-compose up

2. Now since we are using volumes to map, is it not necessary to have in our docker file 'COPY . .'
    - Well you may want to leave that as in the future you may want to not use docker compose or volumes at all


5 CREATE DOCKER CONTAINER FOR TEST APP IN DEV
-------------------------------------------

1.  So in order to run the test you need to create another service in the docker-compose file to also read from the local directory using volumes so each time you change the test (available in src/App.test.js) you can see the test being updated in the console, also we added another command to overwrite the one in Dockerfile.dev

2. You have to add 'command: ["npm","run","test"]' for that container to overwrite the command which is in dockerfile.dev, also notice how for this test service there is no port, it is not needed as it does not test on web.

3. Run it with: docker-compose up --build
    - Now each time you add a new test in app.test.js it will automatically run
    - Note: there is an issue on Windows that the test is not being refreshed in console


6 CREATE DOCKER CONTAINER FOR APP IN PROD ENV
---------------------------------------

1. Now we need to run our app in production, for that we are going to use NPM RUN BUILD
    - Basically it takes all the JS file, put them together and spits them out in a folder in the harddrive
    - In order to communicate our files with users browser, we need a web server (as the localhost for dev server does not exist at this point), something light and very useful, so we are going to use NGINX

2. Create a separate dockerfile to put in production our web app: Dockerfile

3.  Now the thing is that base image needs node, but we also need a base image wit Nginx. **To overcome that we are going to use multi-steps build**

![Image description](https://github.com/jorgeautomation/docker-react/blob/master/image-multiplestep.png)

4. we create a dockerfile where the first stage is to run npm build to get our files and then a second base image to get the files from that build (app/build) and put them into '/usr/share/nginx/html' which is the default place for nginx for static content
    - **you don't have to put a command to start or run nginx, it will run once the container is created**

5. Run 'docker build .' to create that image for the build

- Finally run the container with 'docker run -p 8080:80 <image id>'
    - Navigate to localhost:8080 and boom!!! you are all set!!

7 CONTINUOUS INTEGRATION WITH GITHUB
----------------------------------

- Create a new repository in GitHub (something like docker-react, as public)

- Go to the folder where the project is and:
    - git init
    - git add .
    - git commit -m "initial commit
    - git remote add origin https://github.com/jorgeautomation/docker-react.git
    - git push origin master

CONTINUOUS DELIVERY WITH TRAVIS CI
----------------------------------

- Go to https://travis-ci.org/ and sign in with Github

- Go to your profile, switch on your 'docker-react', then go to the dashboard

- We have to create a .travis.yml file to tell Travis what we want to do with our repository
    - Create the file in root folder
    - sudo: required to run it as administrator
    - create a docker service to have a version of Docker ready to use
    - put the Dockerfile.dev to be installed, we have to put it a tag because since this is running automatically we cannot take the image created for our containers.
    - add the script to run, remember first our test. We have to add '-- --coverage' because if not travis will not never return the control to the command (because it will wait more instructions for testing)

- We will commit the changes to Gitgub, as soon as we do that, since we now have a .travis.yml will build the image and test it and report to us.

AWS ELASTIC BEANSTALK
---------------------

- Now we will be host our application to any cloud service, in this case AWS

- Go to https://aws.amazon.com/ and login to your account and look for Elastic Beanstalk, which is the easiest way to run a single container

- Go to create new application -> add a name -> create environment -> choose web server environment -> on base configuration, platform choose Docker -> create environment

- The benefits of amazon Beanstalk is that automatically scale if the traffic is too much, using the load balancer that is created automatically. 

- Once done you can put the URL in any browser and see your app

- NOTE: REMEMBER TO DELETE your beanstalk once you do this exercise to avoid paying real money for it, go to Actions -> delete application (all in 'All applications' dashboard)

LINKING TRAVIS TO AWS
---------------------

- Now we change the .travis.yml to take into conderation the deploy section
    - provider will be elasticbeanstalk
    - the regions depends on the one of your amazon beanstalk
    - app would be the name you put it
    - env would be the name of your env
    - the bucket_name, you have to go so S3 to see the name of the bucket that was created with your elastic beanstalk
    - bucket_path will be your app name

- Then set the keys for security
    - Go to AWS -> in services look for AIM -> go to Users -> add user -> choose a name (and add programmatic access only)
    - Attach existing policies or permissions -> look for beanstalk and select the one with full access

- Remember that secret key will be shown just ONCE, if you want to see it again you will have to regenerat it.

- We are not going to put the keys in our travis file directly as the github repo is PUBLIC, instead 
    - Go to travis tor your project -> more options -> settings -> environment variables
    - AWS_ACCESS_KEY : "value from aws"
    - AWS_SECRET_KEY : "value from aws"

MAPPING THE PORT IN AWS
-----------------------

- Now if you go to the S3 bucket you will see the folder of the project and in the Beanstalk you will see it is deploying but not working. The reason is that we haven't specified the external port in the dockerfile for production

- We have to add 'EXPOSE 80' in our dockerfile, so AWS will look for



CREATE FINAL FLOW WITH PULL REQUESTS
------------------------------------

- Create a new branch with 'git checkout -b feature' to create and locate in that 'feature' branch

- Do some change in any file -> 'git add ' -> 'git commit -m "changed app text"' -> 'git push origin feature'

- Go to Github, yo the new branch and click on 'Commit and pull request'
    - You will notice two notifications from Travis about code being executed.
    - Now that the build is succedded Click on 'Merge pull request' and the build in Travis
    will happen BUT this time it will deploy to AWS as master is the branch we had to run in AWS

- That's it what is how we finish our proyect

- NOTE: REMEMBER TO DELETE your beanstalk once you do this exercise to avoid paying real money for it, go to Actions -> delete application (all in 'All applications' dashboard)

