
# How to Build a Flask, ReactJS Application and Deploy to Okteto
## Introduction 

**[Flask](https://flask.palletsprojects.com/en/2.0.x/)** is an open source web microframework built with Python. It is straightforward, easy to understand, and is used to build web applications and also APIs. 

In this article, you'll be building a TODO list application directly in Okteto using the [Okteto Development Environment](https://okteto.com/docs/reference/development-environment/) with the Flask framework and ReactJS library.
 
Remote development with Okteto doesn't require an active docker session. Instead, we provide you a persistent volume, instant file synchronization, and an accessible public endpoint to view your application changes as you develop your application locally.
<!-- more -->

**[ReactJS](https://reactjs.org/)** is an open-source front-end JavaScript library used to build user interfaces.

**[Okteto](https://okteto.com/)** is an open-source tool that runs locally to synchronize application code changes to a running pod in a local or remote Kubernetes cluster. With Okteto you're able to see a deployment version of your app as you're developing it.

Instead of working locally, you'll use Okteto's development environment. Although your application can be built locally in your machine, using Okteto's development environment enables you to develop your application in a production ready environment.

## Prerequisities

In order to follow this article, the following is required:

- Basic understanding of Python and the [Flask](https://flask.palletsprojects.com/en/2.0.x/) framework
- Basic understanding of [ReactJS](https://reactjs.org/)
- Basic understanding of [Docker](https://docker.com/)
- [NPM](https://npmjs.org/) installed

## Build Flask API

The first step is to create a project directory to house our todo list application:

```console
$ mkdir flask-react-app
$ cd flask-react-app
```

Next, create the folder for the API application and create a route file:

```console
$ mkdir api && cd api
$ touch todo_list_api.py
```

Next, create a `requirements.txt` file where your application dependencies will be listed. In this case, you only have one, which is `Flask`:

```txt
Flask==1.1.2
Flask-SQLAlchemy==2.5.0
```

[Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com/en/2.x/) will be used to handle the database in which the todo data will be stored.

Next, you're going to create a `Dockerfile`. This Dockerfile will contain the instructions for building your application's image:

```console
$ touch Dockerfile
```

Add the following to it:

```Dockerfile
FROM python:3.9

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

## Scaffolding the frontend application

For the frontend part of this application, you'll be using [Create-React-App](https://reactjs.org/docs/create-a-new-react-app.html/) to scaffold a new application.

In the project directory, `flask-react-app`, run the command to create a frontend application in React:

```console
$ npx create-react-app frontend
$ cd frontend
```

The Okteto development environment listens to the port `8080`. Add a proxy and change the port value in `start` under the `scripts` section in your  `package.json` file:

```json
"proxy": "http://api:5000",
"scripts": {
    "start": "PORT=8080 react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
},
```

Install additional dependencies for the application:

```console
$ npm i react-router react-router-dom web-vitals
```

Next, you're going to create a `Dockerfile` for your frontend application:

```console
$ touch Dockerfile
```

Add the following to your Dockerfile:

```Dockerfile
 
FROM node:14.17.3

WORKDIR /usr/src/app 

COPY ./package.json ./

COPY ./package-lock.json ./

RUN npm install 

COPY . .

EXPOSE 8080

ENTRYPOINT [ "npm" ]

CMD ["start"]
```

Create a `docker-compose.yml` file so you can run both services at once.

In the `docker-compose.yml` manifest, you're going to define two services: `api` for the backend and `web` for the frontend. The `web` service is dependent on the `api` service.

Add the following to your `docker-compose.yml` manifest:

```yaml
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
      - ./frontend:/usr/src/app
    depends_on:
      - api

```

Now run the following commands to build and start your application:
```console
$ docker-compose build
$ docker-compose up
```

Your React application is now running and can be accessed at [http://localhost:8080](http://localhost:8080)

![Frontend Application](localhost.png)

## Build application in Okteto

First, you have to [install  Okteto CLI](https://okteto.com/docs/getting-started/installation/) to access Okteto's development environment.

Next, log in to Okteto Cloud from your CLI:

```console
$ okteto login
```

You'll be prompted to your browser. If it's successful, you'll see something similar to this:

![Login Successful](okteto-welcome.png)

Next, run `okteto namespace` to switch your context and download your Kubernetes credentials.

Now let's deploy the application using the `docker-compose.yml` file you created earlier:

```console
$ okteto stack deploy
```

After executing that, your docker-compose file will be deployed and built on Okteto cloud.

Go to [Okteto cloud](https://cloud.okteto.com/) and you'll see that your application has been deployed successfully:

![image 1](dashboard.png)

The endpoints for your application have been automatically generated.

## Set up your Okteto development environment for Flask

In your terminal, create an Okteto manifest in your `api` folder:

```console
$ okteto init
```

After running it, you'll be prompted to select the deployment you want to develop and select "use default".

You'll see that `okteto.yml` and `.stignore` were created at the root of your project.

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

You'll have to change the `forward` route to `5000` because the frontend will be using port `8080`:

```yml
forward:
  - 5000:5000
```


### Description of the files created

#### Okteto.yml

`okteto.yml` tells Okteto CLI how your project's development environment should be run.
-   `name`: the name of your project.
-   `command`: handles the command that is used to run your project.
-   `sync`: this handles the folders that will be synchronized between your local machine and the development container.
-   `forward`: used to provide a list of ports to forward from your development container.
-   `reverse`: a list of ports to connect development container to your local machine.
-   `volumes`: paths in your development container to be mounted as persistent volumes.
-   `sync`: the folders that will be synchronized between your local machine and the development container.


#### .stignore

`.stignore` tells Okteto CLI which files not to synchronize to your development environment. If you're familiar with `.gitignore`, `.stignore` has a similar function.


Next, activate your development container:

```
$ okteto up 
```


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

Now install the requirements:

```
pip install -r requirements.txt 
```

Now your development environment is currently running on Okteto cloud.

Remember the `todo_list_api.py` file you created earlier? Navigate to it and add the following code:

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


The code above contains the routes and functions that performs the CRUD operations in the todolist application.

The `#` before the command symbolizes that you're working with the development environment provided by Okteto.

To test your API, add a todo. To do that, start the `python` shell in your development container terminal.

Next, run the following commands to create an `example.db` file in your project:

```shell
> from todo_list_api import db
> db.create_all()
```

Next, run the following commands to add a todo list entry:

```shell
> from todo_list_api import TodoList
> todo = TodoList(detail="Learn Okteto")
```

Save the todo entry:

```shell
> db.session.add(todo)
> db.session.commit()
```

Exit the Python shell with `ctrl + d`

Start your Flask server by running `python todo_list_api`. You'll be able to access your API from the URL for your API provided by Okteto.


![API](api.png)

### Set up your Okteto development environment for React

In another terminal window, navigate to your project's directory root and initialize an Okteto manifest:

```console
$ okteto init
```

Select the web deployment:

```console
Select the deployment you want to develop:
  ▸ web
  ▸ api
  Use default values
```

You'll notice an `okteto.yml` and `.stignore` file have been created in the root of your project.

Next, activate your development container:

```console
$ okteto up
```

```
 ✓  Images successfully pulled
 ✓  Files synchronized
    Context:   cloud_okteto_com
    Namespace: khabdrick
    Name:      web
    Forward:   8080 -> 8080
    Reverse:   9000 <- 9000

Welcome to your development container. Happy coding!
```

### Build the front end

In `frontend/src/components` folder, create a file `form.js`. This file will house the `Form` component responsible for form handling:

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
                <input type='submit'></input>
            </form>
        </>
    )
}
```

In the same folder, create a file `cards.js` to handle the rendering of todos.

Add the following to it:

```jsx
import React from "react";
import {Link} from "react-router-dom"

export const Card = ({ listOfTodos }) => {
    return (
        <>
            {listOfTodos.map(todo => {
                return (

                    <ul key={todo.id}>


                        <li><Link to={`${todo.id}`}>{todo.detail}</Link></li>
                    </ul>
                );
            })}
        </>
    );
};
```
Nevermind the `listOfTodos` prop, you will create it next.

In the same folder, create a new file`TodoList.js`. This file houses the components responsible for retrieving and sending todos from the API:

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

Next, create the detail page, this page will display each todo.

First create a new file  `detailView.js` in `src/components` folder, and add the following:

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

Moving forward, let's update `App.js` so that we can test out what we have.

```jsx
import React from "react";
import "./App.css";
import { TodoPage } from "./components/TodoList";
import { BrowserRouter as Router, Switch, Route } from "react-router-dom";
import { Detail } from "./components/detailView";

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

In your Okteto dashboard, select the second endpoint:

![Okteto UI](dashboard.png)

You should see "Invalid Host header" on the browser. This is because you have not permitted Okteto to run your React application.

To fix this, you'll create a `.env` and disable host check. In the frontend folder, run the command from your terminal:

```console
$ touch .env
```

Add the following to your `.env` file:

```env
DANGEROUSLY_DISABLE_HOST_CHECK=true
```

While doing all these updates, Okteto automatically synchronizes the changes with your application deployed on Okteto.

Next, update your application style in `App.css`:

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

In your okteto development container, start your application:

```console
npm run start
```

You'll see that you can view and create todos.

![Todos](todolist.png)

Next, you'll build the update and delete functionality. In the `src/components` folder create a new file `deletePage.js`.

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

For `update` you'll have to make some changes to the detail view, and add a form to use for updating. Update your `detailView.js` file to look like below.

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
Now click on a particular todo, and you should see something like this:

![Todo](todo.png)

## Conclusion

In this article, you learned how to build a Flask and React application by developing directly in Okteto's development environment. You also learned how to deploy docker-compose applications to Okteto. While you could develop locally, by developing directly in Okteto's development environment, you can avoid having to install Docker locally, and can develop your app in a production-like environment. You can find the code used in this article on [GitHub](https://github.com/okteto/flask-react-app/). 

Have any questions about Okteto? Come join the conversation on the #okteto channel in the [Kubernetes Slack workspace](http://slack.k8s.io). 
