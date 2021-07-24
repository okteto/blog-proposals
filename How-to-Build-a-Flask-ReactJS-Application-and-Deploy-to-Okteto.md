
# How to Build a Flask, ReactJS Application and Deploy to Okteto
## Introduction 

**[Flask](https://flask.palletsprojects.com/en/2.0.x/)** is a great open source web microframework built with Python. It is straightforward and easy to understand.
Flask is used to build web applications and also APIs. In this article you will use it to build a todo list API.

**[ReactJS](https://reactjs.org/)** is an open-source  front-end  JavaScript library used to build user interfaces.

**[Okteto](https://okteto.com/)** is an open-source tool that runs locally to synchronize application code changes to a running pod in a local or remote Kubernetes cluster. With Okteto you are able to see deployment version of your app as you are developing it.
Okteto has a development environment which you are going to use instead of working locally. Of course you can build this project with your development but you’ll be using Okteto's development environment to build your app which gives us a production-like development environment.

In this article you will build a todo list application with React and Flask and also build and deploy with Okteto.

### Prerequisities
- Basic understanding of Flask
- Basic understanding of ReactJS
- Basic understanding of Docker


## Build Flask API
In this article, I will assume you have Python already installed on your local device.
Now create a folder for your project `flask-react-app` and navigate to that folder on your terminal/command line `$ cd flask-react-app`.
Next, create a file for your API in `flask-react-app/api/todo_list_api.py`.


Create `requirements.txt` file which you will use to list your dependencies. In your case you only have one and that is Flask.
```
Flask==1.1.2
Flask-SQLAlchemy==2.5.0
```
[Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/en/2.x/) will be used ho handle the database in which the todos will stored.
Now you are going to create a `Dockerfile`. This file will contain all the commands you want to call when building your image.

```Dockerfile
FROM python:3.8.2

WORKDIR /api

ENV FLASK_ENV=development

COPY ./requirements.txt .

RUN pip install -r requirements.txt

COPY . .

CMD [ "python", "todo_list_api.py" ]

```
Your file structure should now look like this:
```
├── flask-react-app
│   ├── api
│   │   ├── todo_list_api.py
│   |   ├── Dockerfile
│   |   ├── requirements.txt		

```

## Create-react-app
You are going to create a react project using create-react-app.
I am going to assume you have npm installed on your local device. That said, navigate to `flask-react-app` folder and run
`$ npx create-react-app frontend`. 
After it runs completely, some files will be created for you. 

Since Okteto requires your application to start with port `8080` you will change it in your scripts. 
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
Now you are going to create `Dockerfile` for your docker application.
On your text editor, navigate to `frontend` folder and create `Dockerfile` and input the code:

```Dockerfile
 
FROM node:14.17.3

WORKDIR /user/src/app 

COPY ./package.json ./

COPY ./package-lock.json ./

RUN npm install 

COPY . .

EXPOSE 8080

ENTRYPOINT [ "npm" ]

CMD ["start"]

```

I know you would like to test your code, but let's first create your `docker-compose.yml` file so that you can run both servers at once.
With the `docker-compose.yml` commands you will connect the images created for React and Flask and run them together. You will also use the `docker-compose.yml` to set how you would like Okteto to run your application.
In `flask-react-app` folder create `docker-compose.yml` file and put the code:

```yml
services:
  api:
    build:  ./api
    ports: 
    - "5000:5000"
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

You'll see that your react application is running on port `localhost:8080`


##Build Application in Okteto
First, you have to [install  Okteto CLI](https://okteto.com/docs/getting-started/installation) to access Okteto's development environment.

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

Now let's deploy the application using the `docker-compose.yml` file you created:

`$ okteto stack deploy` 
After executing that your docker-compose file will be deployed and built on Okteto cloud.
Go to [Okteto cloud](https://cloud.okteto.com/) and you'll see that your application has been deployed successfully.
 
 Clicking on the second link will take you to the home page of your react application.
  ![image 1]()
![enter image description here](sss)
### Set up your Okteto Development Environment for Flask
On your terminal, navigate the directory that contains your API file then run `$ okteto init` to initialize Oketo at the root of your project.

After running it, you will be prompted to select the deployment you want to develop and select "use default".
You will see that `okteto.yml` and `.stignore` was created at the root of your project.
```yaml
#okteto.yml
name: api
autocreate: true
image: okteto/python:3
command: bash
volumes:
- /root/.cache/pip
sync:
- .:/usr/src/app
forward:
- 8080:8080
reverse:
- 9000:9000
```
You will have to change the "foward" route to `5000` because the front end willbe using `8080`
```yml
forward:
- 5000:5000
```
### Description of the Files Created
**Okteto.yml**
`okteto.yml` tells OktetoCLI how your project development environment should be run.
-   `name`: the name of your project.
-   `command`: handles the command that is used to run your project.
-   `sync`: this handles the folders that will be synchronized between your local machine and the development container.
-   `forward`: used to provide list of ports to forward from your development container.
-   `reverse`: a list of ports to connect development container to your local machine.
-   `volumes`: paths in your development container to be mounted as persistent volumes.
-   `sync`: the folders that will be synchronized between your local machine and the development container.


**.stignore**
`.stignore` tells Okteto CLI which files not to synchronize to your development environment. If you are familiar with `.gitignore`, it has a similar function to that.


Next, you have to activate your development environment, to do that run:
```
$ okteto up 
```
You should see:
```

 ✓  Images successfully pulled
 ✓  Files synchronized
    Context:   cloud_okteto_com
    Namespace: khabdrick
    Name:      api
    Forward:   5000 -> 5000
    Reverse:   9000 <- 9000

Welcome to your development container. Happy coding!
```
Now install the requirements, there are only 2
```
pip install -r requirements.txt 
```
Now your development environment is currently running on Okteto cloud. 
Next, navigate to your application
Next, let's build your application. Remember the `todo_list_api.py` file you created earlier?
Navigate to it and put the following code:
```python
from flask import Flask, jsonify, request, json
from flask_sqlalchemy import SQLAlchemy

#instantiate Flask functionality
app = Flask(__name__)

# set sqlalchemy URI in application config
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///example.db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
# instance of SQL

db = SQLAlchemy(app)


class TodoList(db.Model):
# This class will handle the models for the todo list
    id = db.Column(db.Integer, primary_key=True)
    detail = db.Column(db.Text, nullable=False)

    def __str__(self):
      return f"{self.id} {self.detail}"

def todo_serializer(todo):
# covert the object coming from the database into JSON
  return{
    "id":todo.id,
    "detail":todo.detail
  }

@app.route('/api', methods=['GET'])
def home():
# Display all the todos in the database
  return jsonify([*map(todo_serializer, TodoList.query.all())])

@app.route('/api/todocreate', methods=['POST'])
def todo_create():

  request_data = json.loads(request.data)
  todo = TodoList(detail=request_data['detail'])

  db.session.add(todo)
  db.session.commit()

  return{'201':'todo created successfully'}

@app.route('/api/<int:id>')
def detail_view(id):
# Detail view of each todo
  return jsonify([*map(todo_serializer, TodoList.query.filter_by(id=id))])

@app.route('/api/<int:id>', methods=['PUT'])
def update_todo(id):
# Updates the todo with the id
  todo = TodoList.query.get(id)
  detail = request.json['detail']
  todo.detail = detail
  db.session.commit()

  return {"200":"Updated successfully"}

@app.route('/api/<int:id>', methods=['POST'])
def delete_todo(id):
# Deletes todo from the database based on id
  request_data = json.loads(request.data)
  TodoList.query.filter_by(id=request_data['id']).delete()
  db.session.commit()
  return {"204":"Updated successfully"}

if __name__ == '__main__':
  app.run(debug=True, host='0.0.0.0')
```
The code above is just an API that performs the CRUD functionalities on the todolist application. I added comments to the code to aid your understanding.
`#` before the command symbolizes you are working with development envitonment provide by Okteto
Next, Install Flask requirements:
`# python install -r requirements.txt`
To test your API let's add a todo. To do that type `python` and enter, this will enable you type Python script on the terminal.
Next, run `from todo_list_api import db` then `db.create_all()` which will create `example.db` file in your project.
Now, run `from todo_list_api import TodoList` then run `todo=TodoList(detail="Learn Okteto")`
To save it in the database run `db.session.add(todo)` then run `db.session.commit()`

Next, exit the Python shell with `ctrl + d`
Run your Flask server  `python todo_list_api`. You will now bw able to access your API from the URL for API provided by Okteto. Mine is this: https://api-khabdrick.cloud.okteto.net/, so add "api" at the end of the URL will display the todo you just saved.

  ![image 2]()
  
You can explore the other functionalities on your own.

### Set up your Okteto Development Environment for React 

Open another terminal and navigate to the root of your project then run `okteto init` select "web"  so Okteto will create an Okteto file based on your `docker-compose.yml` file.
You'll notice an `okteto.yml` and `.stignore` file have been created in the root of your project.
Next run  `okteto up`

### Build the Front End
To build your application,  `frontend` > `src` and create a file `form.js` this file will handle the form you will use to store you. the content of the file will be:
```jsx
import React from "react"


export const Form = ({input, onFormChange, onFormSubmit}) => {

    const handleChange = (event)=>{
     // handle what the form does when you type in it
        onFormChange(event.target.value)
    }

    const handleSubmit = (event)=>{
    // handle what the form does when it is submitted
        event.preventDefault()
        onFormSubmit()
    }

    return(
        <>
        <form onSubmit={handleSubmit}>
            <input className='form-class' type='text' required value={input} onChange={handleChange}></input>
            <input  type='submit'></input>
        </form>
        </>
    )
}
```
Still in the `src/components` folder create a file to handle to map through the list of todos in the database. The file you will create is `card.js`, then put the code below:
```jsx
import React from "react";
import {Link} from "react-router-dom"

export const Card = ({ listOfTodos }) => {
  return (
    <>
      {listOfTodos.map(todo => {
          // console.log(todo)
        return (
            
          <ul key={todo.id}>
              
            <li className="links"><Link to={`${todo.id}`}>{todo.detail}</Link></li>
          </ul>
        );
      })}
    </>
  );
};
```
Nevermind the `listOfTodos` prop, you will create it next.
Create a new file in the components folder were you will be fetching the the data from the API.
`TodoList.js`
```jsx
import React, { useState, useEffect } from "react";
import { Card } from "../components/card";
import { Form } from "../components/form";

export const TodoPage = () => {
  const [todo, setTodo] = useState([]);
  const [addTodo, setAddTodo] = useState("");

  useEffect(() => {
  // Fetch the list of todos API
    fetch("/api")
      .then((response) => {
        if (response.ok) {
          return response.json();
        }
      })
      .then((data) => setTodo(data));
  }, []);

  const handleFormChange = (inputValue) => {
    setAddTodo(inputValue);
  };

  const handleFormSubmit = () => {
  // Creates todo suing the API
    fetch("/api/todocreate", {
      method: "POST",
      body: JSON.stringify({
        detail: addTodo,
      }),
      headers: {
        "Content-type": "application/json; charset=UTF-8",
      },
    })
      .then((response) => response.json())
      .then((message) => {
        console.log(message);
        setAddTodo("");
        getUpdate();
      });
  };

  const getUpdate = () => {
  // Automatically update the list as you submit the form
    fetch("/api")
      .then((response) => {
        if (response.ok) {
          return response.json();
        }
      })
      .then((data) => setTodo(data));
  };

  return (
    <>
      <Form
        input={addTodo}
        onFormChange={handleFormChange}
        onFormSubmit={handleFormSubmit}
      />
      <Card listOfTodos={todo} />
    </>
  );
};

```
Straight foward enough!
Now let's create the detail page, this page will display each todo.
First create a new file  `detailView.js` in src/components folder and put the code below.
```jsx
import React, { useState, useEffect } from "react";
import { Link, useParams } from "react-router-dom";
import { useHistory } from "react-router-dom";

export const Detail = () => {
  const history = useHistory();
  const [todo, setTodo] = useState([]);
  const { id } = useParams();

  useEffect(() => {
    fetch(`/api/${id}`)
      .then((response) => response.json())
      .then((data) => setTodo(data));
  }, [id]);

      .then((response) => response.json())
      .then((message) => {
        history.push("/"); //redirect to homepage

      })
  
  };

  return (
    <div>
       <br/>
      {todo.map((data) => (
        <div key="id">Detail: {data.detail}</div>
      ))}
      <br />

      <Link to="/">Go Back to Todo List</Link>
    </div>
  );
};

```

Moving foward, let's update `App.js` so that we can test out what we have.
```jsx
import React from "react";
import "./App.css";
import { TodoPage } from "./components/TodoList";
import { BrowserRouter as Router, Switch, Route } from "react-router-dom";
import { Detail } from "./components/detailView";
// import { Form } from './components/form';

function App() {
  return (
    <div className="App">
      <header className="App-header">
        <Router>
          <Switch>
            <Route exact path="/">
              <TodoPage />
            </Route>
            <Route path="/:id">
              <Detail />
            </Route>
          </Switch>
        </Router>
      </header>
    </div>
  );
}

export default App;
```
There are some tools I used but haven't installed yet, so what you will do is update your `package.json` file to look like below:
```json
{
  "name": "frontend",
  "version": "0.1.0",
  "proxy": "http://api:5000",
  "private": true,
  "dependencies": {
    "@testing-library/jest-dom": "^5.14.1",
    "@testing-library/react": "^11.2.7",
    "@testing-library/user-event": "^12.8.3",
    "react": "^16.13.1",
    "react-dom": "^17.0.2",
    "react-router": "5.2.0",
    "react-router-dom": "^5.2.0",
    "react-scripts": "4.0.3",
    "web-vitals": "^1.1.2"
  },
  ...
```
Because you are working on Okteto cloud your changes are synchronized automatically. So what you will do now is, on your terminal navigate to the folder where `package.json` and run `npm install` when that is done run `npm start` 
NB: You must be in okteto development container (`okteto up`)

Go to Okteto cloud, there are 2 endpoints displayed on, click on the one that starts with "web".

 You should see "Invalid Host header" on the browser. This is because you have not permitted Okteto to run your React application.
  
To fix this, you will create a `.env` and disable host check.
Run `$ touch .env` in the `frontend` folder.
In the file you just created, put the statement:
`DANGEROUSLY_DISABLE_HOST_CHECK=true` .

Update your `App.css` file to contain this, just to add a little styling:
```css

*{
    text-decoration: none;
} 

body {
  font-size: 1.6rem; 
  background-color: #efefef;
  color: #324047
}


.App-header{
padding-left: 500px;
}

.form-class{
margin-top: 100px;
}

.links{
  position: relative;
  list-style-type: square;
  padding-left: 1rem;
  margin-bottom: 0.5rem;
}

```

While doing all these updates, Okteto is synchronizing the changes with your application deployed on Okteto.
Now run `$ npm start` to start the server and go back to the URL  you got from Okteto cloud home page. You'll see that you can view and create todos.


  ![image 3]()

Next, you will build the update and delete functionality. In the `src/components` folder create a new file `deletePage.js`.

```jsx
import React from "react"
import {useHistory} from 'react-router-dom'

export const Delete = ({id}) =>{
    const history = useHistory()
    const deleteTodo = () =>{
        fetch(`/api/${id}`,{
            method: 'POST',
            body:JSON.stringify({
                id:id
            })
        }).then(response => response.json())
        .then(data => {
            history.push('/')

        })

    }

    return(
        <>
        <button onClick={deleteTodo}>Delete</button>
        </>
    )
}
```

The code above is just fetching the delete function and applying it to the `delete` button. 
For `update` you will have to make some changes to the detail view and add a form to use for updating. So what you have to do ids update your `detailView.js` file to look like below.

```jsx
import React, { useState, useEffect } from "react";
import { Link, useParams } from "react-router-dom";
import { Delete } from "../components/deletePage";
import { Form } from "../components/form";
import { useHistory } from "react-router-dom";

export const Detail = () => {
  const history = useHistory();
  const [todo, setTodo] = useState([]);
  const [updateTodo, setUpdateTodo] = useState("");
  const { id } = useParams();

  useEffect(() => {
    fetch(`/api/${id}`)
      .then((response) => response.json())
      .then((data) => setTodo(data));
  }, [id]);

  const handleFormChange = (inputValue) => {
    setUpdateTodo(inputValue);
  };

  const handleFormSubmit = () => {
    fetch(`/api/${id}`, {
      method: "PUT",
      body: JSON.stringify({
        detail: updateTodo,
      }),
      headers: {
        "Content-type": "application/json; charset=UTF-8",
      },
    })
      .then((response) => response.json())
      .then((message) => {
        history.push("/");

      })
  
  };

  return (
    <div>
      <Form
        input={updateTodo}
        onFormChange={handleFormChange}
        onFormSubmit={handleFormSubmit}
      />
       <br/>
      {todo.map((data) => (
        <div key="id">Detail: {data.detail}</div>
      ))}
      <br />
      <Delete id={id} /><p>Fill the above to update</p>
      &nbsp;&nbsp;
      <Link to="/">Go Back to Todo List</Link>
    </div>
  );
};

```
Now click on the particular todo and you should see something like this:

  ![image 4]()
## Conclusion
In this article, I took you on how to build and run a Flask and React application with Docker and deploy it easily to Okteto. Deploying you application to Okteto can help you see what your application will look like during deployment as you are building or making changes. It will also free up your local machine since you will be running the server on the cloud. From this tutorial, I am hopeful that you will be able to work with Okteto in your current or future Flask and React projects.
You can find the code used in this article on GitHub.
