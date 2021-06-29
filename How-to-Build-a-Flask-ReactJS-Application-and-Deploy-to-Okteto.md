# How to Build a Flask, ReactJS Application and Deploy to Okteto
## Introduction 

**[Flask](https://flask.palletsprojects.com/en/2.0.x/)** is a great open source web microframework built with Python. It is straightforward and easy to understand.
Flask is used to build web applications and also APIs. In this article we will use it to build an API that displays date and time.

ReactJS is an open-source  front-end  JavaScript library used to build user interfaces.

**[Okteto](https://okteto.com/)** is an open-source tool that runs locally to synchronize application code changes to a running pod in a local or remote Kubernetes cluster. With Okteto we are able to see deployment version of our app as we are developing it.
Okteto has a development environment which we are going to use instead of working locally. Of course we can build this project with our development but we’ll be using Okteto's development environment to build our app which gives us a production-like development environment.

In this article we will build an application that displays the date and time with React and Flask and deployment will be handled with Okteto

### Prerequisities
- Basic understanding of Python
- Basic understanding of ReactJS
- Basic understanding of Docker


## Build Flask API
In this article I will assume you have Python already installed on your local device.
Now create a folder for our project `flask-react-app` and navigate to that folder on your terminal/command line `$ cd flask-react-app`.
Next, create a file for our API in `flask-react-app/api/datetime_api.py` and input the code.

```python
from  datetime  import  datetime
from  flask  import  Flask

app = Flask(__name__)

# API that returns the date and time
@app.route('/datetime', methods=['GET'])
def  get_current_time():
	return {'datetime': datetime.now()}

if  __name__ == '__main__':
	app.run(debug=True, host='0.0.0.0')
```
Create `requirements.txt` file which we will use to list our dependencies. in our case we only have one and that is Flask.
```
FLASK==1.1.2
```
Now we are going to create a `Dockerfile`. This file will contain all the commands we want to call when building our image.

```Dockerfile
FROM  python:3.7-alpine

WORKDIR  /api
ENV  FLASK_APP=datetime_api.py

ENV  FLASK_ENV=development
COPY  ./requirements.txt  .

RUN  pip  install  -r  requirements.txt
COPY  .  .

CMD  [  "python",  "datetime_api.py"  ]
```
Your file structure should now look like this:
```
├── flask-react-app
│   ├── api
│   │   ├── datetime_api.py
│	│	├── Dockerfile
│	│	├── requirements.txt		

```

## Create-react-app
We are going to create a react project using create-react-app.
I am going to assume you have npm installed on your local device. That said, navigate to `flask-react-app` folder and run
`$ npx create-react-app frontend`. 
After it runs completely, some files will be created for you. Open your text editor and navigate to `fronend/src/app.js` and put the code below:

```jsx
import  React, { useState, useEffect } from  'react';
import  './App.css';

function  App() {
	// Used to initialize the state
	const [currentDateTime, setCurrentDateTime] = 		useState(0);
	const  url = '/datetime'
// fetch data from API
useEffect(() => {
fetch(url).then(res  =>  res.json()).then(data  => {
setCurrentDateTime(data.datetime);

});
}, []);  

return (
	<div  className="App">
		<header  className="App-header">
			<h3>
		The date and time is {currentDateTime}
			</h3>
		</header>

		</div>

	);

}

export  default  App;
```
Since Okteto requires our application to start with port `8080` we will change it in our scripts. 
In your text editor, navigate to `frontend/package.json`
```json
...
"scripts": {
	"start": "PORT=8080 react-scripts start",
	"build": "react-scripts build",
	"test": "react-scripts test",
	"eject": "react-scripts eject"
},
...
```
Now we are going to create `Dockerfile` for our docker application.
On your text editor, navigate to `frontend` folder and create `Dockerfile` and input the code:
```Dockerfile
FROM  node:14-slim
WORKDIR  /user/src/app

COPY  ./package.json  ./

COPY  ./package-lock.json  ./

RUN  npm  install
COPY  .  .
EXPOSE  8080
ENTRYPOINT  ["npm"]
CMD  ["start"]

```

I know you would like to test your code, but let's first create our `docker-compose.yml` file so that we can run both servers at once.
With the `docker-compose.yml` commands we will connect the images created for React and Flask and run them together. We will also use the `docker-compose.yml` to set how we would like Okteto to run our application.
In `flask-react-app` folder create `docker-compose.yml` file and put the code:

```yml
services:
	api:
		build: ./api
		ports:
		- "90:5000"
		volumes:
		- ./api:/api
		environment:
			FLASK_ENV: development
	web:
		build: ./frontend
		ports:
			- "8080:8080"
		volumes:
			- ./frontend:/user/src/app
		depends_on:
			- api
```
Now run `$ docker-compose build` and then `$ docker-compose up`


You'll see that our react application is running on port `localhost:8080`

![](First-image)
## Deploying to Okteto
First, you have to install the  [Okteto CLI](https://okteto.com/docs/getting-started/installation) to access Okteto's development environment.

Next, let’s login to Okteto Cloud, to do that run 
`$ okteto login`
You will be prompted to your browser. If it is successful you will see something similar to this:
```
Authentication will continue in your default browser
You can also open a browser and navigate to the following address:
https://cloud.okteto.com/auth/authorization-code?redirect=http%3A%2F%2F127.0.0.1%3A40403%2Fauthorization-code%2Fcallback%3Fstate%3DuWjNxSyWG7rDOkWh0tCvNacXQNBe1O83uvhHyziYSss%3D&state=uWjNxSyWG7rDOkWh0tCvNacXQNBe1O83uvhHyziYSss%3D
 ✓  Logged in as khabdrick
    Run `okteto namespace` to switch your context and download your Kubernetes credentials.
```
Next, Run `okteto namespace` to switch your context and download your Kubernetes credentials.

Now let's deploy the application using the `docker-compose.yml` file we created:

`$ okteto stack deploy`

### Set up your Okteta Development Environment


Now run `$ okteto init` to initialize Oketo at the root of your project

After running it, you will be prompted to select the deployment you want to develop and select "web" which is the frontend service created earlier. since it is what is going to Nevermind the user interface part, we will add the configurations to `okteto.yml` later.

Now you will see that `okteto.yml` and `.stignore` was created at the root of your project.

### Description of the Files Created
**Okteto.yml**
`okteto.yml` tells OktetoCLI how your project development environment should be run.
-   `name`: the name of our project.
-   `command`: handles the command that is used to run our project.
-   `sync`: this handles the folders that will be synchronized between our local machine and the development container.
-   `forward`: used to provide list of ports to forward from your development container.


**.stignore**
`.stignore` tells Okteto CLI which files not to synchronize to your development environment. If you are familiar with `.gitignore`, it has a similar function to that.


Next, we have to activate our development environment, to do that run:
```
$ okteto up 
```
You should see:
```
 ✓  Persistent volume successfully attached
 ✓  Images successfully pulled
 ✓  Files synchronized
    Context:   cloud_okteto_com
    Namespace: khabdrick
    Name:      web
    Forward:   3000 -> 3000
               8080 -> 8080
               9229 -> 9229
```
Now our development environment is currently running on Okteto cloud. 

Next, `frontend# cd frontend` and run 
`frontend# npm install && npm run`
Go to [Okteto cloud](https://cloud.okteto.com/) and you'll see that our application has been deployed successfully.
![](Second-image)
There will be 2 endpoints displayed on the Okteto cloud, click on the one that starts with web.
 You should see "Invalid Host header". This is because we have not permitted Okteto to run our React application.
 ![](Third-image)

We are going to use this fix to test the power of Okteto. 
To fix this, we will create a `.env` and disable host check.
Run `$ touch .env` in the `frontend` folder.
In the file you just created, put the statement:
`DANGEROUSLY_DISABLE_HOST_CHECK=true` .

While doing all these updates, Okteto is synchronising the changes with your application deployed on Okteto.
Now run `$ npm start` to start the server and go back to the URL  you from Okteto cloud home page.
Since everything is synchronised automatically you should see:
![](Fourth-image)



 


## Conclusion
In this article I took you on how to build and run a Flask and React application with Docker and deploy it easily to Okteto. Deploying you application to Okteto can help you see what your application will look like during deployment and also frees up your local machine since you will be running the server on the cloud. From this tutorial, I am hopeful that you will be able to work with Okteto in your current or future Flak and React projects.
You can find the code used in this article on GitHub.
