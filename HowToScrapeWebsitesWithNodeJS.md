Sign in
Get started
Level Up Coding
WRITE FOR US
CODING INTERVIEW COURSE →
You have 2 free stories left this month. Sign up and get an extra one for free.
How to Scrape Websites with Node.js
John Au-Yeung
John Au-Yeung
Follow
Oct 21, 2019 · 12 min read




Web scraping is a technique where you extract data from a website by parsing the HTML. This is done by loading the website’s content into a DOM tree, parsing the elements, extracting data from the desired elements, and storing the results in a database.
In this article, we will make a web scraper that fetches listings from a deals website called Red Flag Deals and stores the data in a database. We will have a backend that manages a queue of scrape requests that we will call “jobs”. We provide the ability add, view, and delete jobs. Then we create a frontend to let us view the data we scraped. The web scraper will be a Node.js script. The backend will be an Express app with Sequelize as the ORM. We will use SQLite as our database to keep things simple for this demonstration app. The frontend will be written with React.
Web Scraper Backend
We will start with the backend. Create a project folder and within it, create a backend folder for our backend code. Next, we run Express Generator by executing npx express-generator. This will create the files we need for the base Express app. Now we install the packages for our backend app by running npm install.
Once the packages are installed, we will add Babel to the app so that we can use features of JavaScript that aren’t available in Node.js such as import.
Run npm i @babel/cli @babel/core @babel/node @babel/preset-env. Next create a file called .babelrc in the backend folder and add the following:
{
    "presets": [
        "@babel/preset-env"
    ]
}
In package.json, we add the following to the scripts section:
"start": "nodemon --exec npm run babel-node --  ./bin/www",
"babel-node": "babel-node"
We start our app with babel-node runtime instead of the normal Node runtime so that we can use Babel. Note that we also need nodemon so that the app will restart when files are changed. To install it, run npm i -g nodemon.
Next we need to install our own libraries. We will use the Bull package for creating a job queue for scraping data, CORS for allowing requests from the frontend, Request and Request Promise for fetching the website text, Sequelize for the ORM, and SQLite3 for storing data. We install the packages by running npm i cors bull cheerio request request-promise sequelize sqlite3.
We also need Redis for our Bull job queue to work. The latest version only runs on Linux so we have to use it. In Ubuntu, we run:
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install redis-server
redis-server
This installs and runs Redis. Detailed instructions are at https://tecadmin.net/install-redis-ubuntu/.
Next we add the code for Sequelize to our backend. Run npx sequelize-cli init. In the config.json, replace the code with the following:
{
  "development": {
    "dialect": "sqlite",
    "storage": "development.db"
  },
  "test": {
    "dialect": "sqlite",
    "storage": "test.db"
  },
  "production": {
    "dialect": "sqlite",
    "storage": "production.db"
  }
}
Then we need to create migrations and models for our database to store our data. Run npx sequelize-cli model:create --name Deal --attributes name:string,description:string,url:string,content:text. Running this will create a migration and the associated model.
When you run npx sequelize-cli db:migrate you will get the Deals table in your SQLite database.
With the database code done, we can start writing the logic. Create a queues folder in the backend folder and add a file called scraperQueue.js and add the following:
const Queue = require("bull");
const cheerio = require("cheerio");
const rp = require("request-promise");
const models = require("../models");
const scraperQueue = new Queue("web scraping");
scraperQueue.process(async (job, done) => {
  const deal = job.data.deal;
  const response = await rp(deal.url);
  const $ = cheerio.load(response);
  const items = $(".list_item");
  let deals = [];
  Object.keys(items).forEach(key => {
    const dealer = $(items[key])
      .find(".offer_dealer")
      .text();
    const title = $(items[key])
      .find(".offer_title")
      .text();
    let link = $(items[key])
      .find(".offer_title a")
      .attr("href");
    link = `https://www.redflagdeals.com${link}`;
    deals.push({ dealer, title, link });
  });
  await models.Deal.update(
    {
      content: JSON.stringify(deals),
    },
    {
      where: {
        id: deal.id,
      },
    }
  );
  done();
});
export { scraperQueue };
This is the queue for our web scraping jobs. What we are doing in this file is loading the web page we want to scrape, which is the Red Flag Deals deals page. We open the response of the page with Cheerio by loading the HTML into the DOM tree, locate the items with the class list-item and then iterate through each element and extract the text from the offer_dealer and offer_title classes. For the link, we load the element into the DOM tree with Cheerio and then get the a element from the node with the class offer_title. Then get the href attribute and add the URL prefix and root URL to it since it’s a relative path.
This means if the page structure changes, for example, when the class names change, then you have to update the code to keep it working.
After getting the data, we save it with Sequelize to the job entry we created.
Next we create the API endpoints to add, get, and delete the data. Create a file called deals.js in the routes folder and add the following:
const express = require("express");
const models = require("../models");
const router = express.Router();
import { scraperQueue } from "../queues/scraperQueue";
router.post("/scrape", async (req, res, next) => {
  const { name, description, url } = req.body;
  const deal = await models.Deal.create({
    name,
    description,
    url,
  });
  scraperQueue.add({ deal });
  res.json(deal);
});
router.delete("/job/:id", async (req, res, next) => {
  const id = req.params.id;
  await models.Deal.destroy({ where: { id } });
  res.json({ id });
});
router.get("/jobs", async (req, res, next) => {
  const deals = await models.Deal.findAll();
  res.json(deals);
});
router.get("/job/:id", async (req, res, next) => {
  const id = req.params.id;
  const deals = await models.Deal.findAll({ where: { id } });
  const deal = deals[0];
  deal.content = JSON.parse(deal.content);
  res.json(deal);
});
module.exports = router;
We have the scrape route to add the job to the queue and save the data to the Deals table. The delete /job/:id route is used to delete data from the Deals table. The /jobs table is for getting the jobs, and the get/job/:id route is for returning the deals data to the user.
Finally we have to make some changes to app.js. We replace the existing code with the following:
const createError = require("http-errors");
const express = require("express");
const path = require("path");
const cookieParser = require("cookie-parser");
const logger = require("morgan");
const cors = require("cors");
const indexRouter = require("./routes/index");
const dealsRouter = require("./routes/deals");
const app = express();
// view engine setup
app.set("views", path.join(__dirname, "views"));
app.set("view engine", "jade");
app.use(logger("dev"));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, "public")));
app.use(cors());
app.use("/", indexRouter);
app.use("/deals", dealsRouter);
// catch 404 and forward to error handler
app.use(function(req, res, next) {
  next(createError(404));
});
// error handler
app.use(function(err, req, res, next) {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get("env") === "development" ? err : {};
// render the error page
  res.status(err.status || 500);
  res.render("error");
});
module.exports = app;
We add the cors add-on to let the app talk to front end, and we added const dealsRouter = require(“./routes/deals”); and app.use(“/deals”, dealsRouter); to expose the deals routes.
This finishes the backend of our web scraping app. Now we need the frontend to add or remove jobs and display the results of the web scraping jobs.
Web Scraper Frontend
We start by creating a React project with Create React App. Create the app by going to the project’s root folder and run npx create-react-app frontend.
Next we need to install some packages. Run npm i axios bootstrap formik mobx mobx-react react-bootstrap react-router-dom yup. Axios is our HTTP client for making requests to our backend. Bootstrap and React Bootstrap are used for styling, MobX and MobX React are for state management, and Formik and Yup are for handling form value changes and form validation respectively.
Now we add and modify files in the src folder unless otherwise mentioned.
First we replace the code in App.js with the following:
import React from "react";
import { Router, Route, Link } from "react-router-dom";
import HomePage from "./HomePage";
import ResultsPage from "./ResultsPage";
import TopBar from "./TopBar";
import { createBrowserHistory as createHistory } from "history";
import { JobsStore } from "./stores";
import "./App.css";
const history = createHistory();
const jobsStore = new JobsStore();
function App() {
  return (
    <div className="App">
      <Router history={history}>
        <TopBar />
        <Route
          path="/"
          exact
          component={props => <HomePage {...props} jobsStore={jobsStore} />}
        />
        <Route path="/results/:id" exact component={ResultsPage} />
      </Router>
    </div>
  );
}
export default App;
We have the TopBar component which displays our top bar which we will create soon. Below that we have the routes for our pages which we will create. Note that we pass in our store to the components in the route. This is the MobX store which will persist the list of jobs for easy access.
In App.css we replace the existing code with:
.center {
    text-align: center;
}
Next we will build the home page. Create a file called HomePage.js and add the following:
import React from "react";
import { useEffect, useState } from "react";
import { Formik } from "formik";
import Form from "react-bootstrap/Form";
import Col from "react-bootstrap/Col";
import Button from "react-bootstrap/Button";
import * as yup from "yup";
import Table from "react-bootstrap/Table";
import { Redirect } from "react-router";
import "./HomePage.css";
import { observer } from "mobx-react";
import { scrape, removeJob, getAllJobs } from "./requests";
const schema = yup.object({
  name: yup.string().required("Name is required"),
  description: yup.string().required("Description is required"),
  url: yup.string().required("URL is required"),
});
function HomePage({ jobsStore }) {
  const [redirect, setRedirect] = useState(false);
  const [jobId, setJobId] = useState(0);
  const [initialized, setInitialized] = useState(false);
  const handleSubmit = async evt => {
    const isValid = await schema.validate(evt);
    if (!isValid) {
      return;
    }
    jobsStore.jobs.push(evt);
    jobsStore.setJobs(jobsStore.jobs);
    await scrape(evt);
    getJobs();
  };
  const selectJob = id => {
    setJobId(id);
    setRedirect(true);
  };
  const getJobs = async () => {
    const response = await getAllJobs();
    jobsStore.setJobs(response.data);
  };
  const deleteJob = id => {
    removeJob(id);
    getJobs();
  };
  useEffect(() => {
    if (!initialized) {
      getJobs();
      setInitialized(true);
    }
  });
  if (redirect) {
    return <Redirect to={`/results/${jobId}`} />;
  }
  return (
    <div className="home-page">
      <h1>Web Scraping Jobs</h1>
      <Formik
        validationSchema={schema}
        onSubmit={handleSubmit}
      >
        {({
          handleSubmit,
          handleChange,
          handleBlur,
          values,
          touched,
          isInvalid,
          errors,
        }) => (
          <Form noValidate onSubmit={handleSubmit}>
            <Form.Row>
              <Form.Group as={Col} md="12" controlId="name">
                <Form.Label>Name</Form.Label>
                <Form.Control
                  type="text"
                  name="name"
                  placeholder="Name"
                  value={values.name || ""}
                  onChange={handleChange}
                  isInvalid={touched.name && errors.name}
                />
                <Form.Control.Feedback type="invalid">
                  {errors.name}
                </Form.Control.Feedback>
              </Form.Group>
              <Form.Group as={Col} md="12" controlId="chatRoomName">
                <Form.Label>Description</Form.Label>
                <Form.Control
                  type="text"
                  name="description"
                  placeholder="Description"
                  value={values.description || ""}
                  onChange={handleChange}
                  isInvalid={touched.description && errors.description}
                />
                <Form.Control.Feedback type="invalid">
                  {errors.description}
                </Form.Control.Feedback>
              </Form.Group>
<Form.Group controlId="url" as={Col} md="12">
                <Form.Label>Website</Form.Label>
                <Form.Control
                  as="select"
                  name="url"
                  value={values.url || ""}
                  onChange={handleChange}
                  isInvalid={touched.url && errors.url}
                >
                  <option value="">Select</option>
                  <option value="https://www.redflagdeals.com/in/toronto/deals/">
                    Red Flag Deals
                  </option>
                </Form.Control>
                <Form.Control.Feedback type="invalid">
                  {errors.url}
                </Form.Control.Feedback>
              </Form.Group>
            </Form.Row>
            <Button type="submit" style={{ marginRight: "10px" }}>
              Add
            </Button>
          </Form>
        )}
      </Formik>
      <br />
      <Table striped bordered>
        <thead>
          <tr>
            <th>Name</th>
            <th>Description</th>
            <th>URL</th>
            <th></th>
            <th></th>
          </tr>
        </thead>
        <tbody>
          {jobsStore.jobs.map((j, i) => {
            return (
              <tr key={i}>
                <td>{j.name}</td>
                <td>{j.description}</td>
                <td>{j.url}</td>
                <td>
                  <Button onClick={selectJob.bind(this, j.id)}>Results</Button>
                </td>
                <td>
                  <Button onClick={deleteJob.bind(this, j.id)}>Delete</Button>
                </td>
              </tr>
            );
          })}
        </tbody>
      </Table>
    </div>
  );
}
export default observer(HomePage);
This page has the form for adding the jobs. We validate that all fields are required with Yup, which we use to create the schema object. We nest our Form component, which is provided by React Bootstrap with the Formik component so that it will handle all the form value changes and return the entered values in the handleSubmit function. We display the form validation errors with the Form.Control.Feedback components. When the user clicks “Add”, the handleSubmit function is executed.
In the handleSubmit function will validate the form values by running the schema.validate function, which returns a promise. Once that is executed and the validation passes, then we call thescrape function in requests.js, which we will create to add the jobs to the queue. After that, it will get the latest jobs by calling getJobs in our component, put the returned entries into our jobsStore and display the items in the table at the bottom.
We need the useEffect callback to call getJobs during the first load of the component so that we can populate the table when the page loads. We do this by checking the initialized flag. If it’s false, the getJobs runs, the initialized will be set to true by calling setInitialized(true) .
We have a button to let us view the jobs with the Result button, and we have a Delete button where we call the removeJob function from requests.js to make the API request to delete the job.
For styling the HomePage component, create HomePage.css and add:
.home-page {
  padding: 20px;
}
Next we writethe code for the requests. Create requests.js and add:
const APIURL = "http://localhost:3000";
const axios = require("axios");
export const getAllDeals = id => axios.get(`${APIURL}/deals/job/${id}`);
export const getAllJobs = () => axios.get(`${APIURL}/deals/jobs`);
export const scrape = data => axios.post(`${APIURL}/deals/scrape`, data);
export const removeJob = id => axios.delete(`${APIURL}/deals/job/${id}`);
Next we create a page to display the results. Create a file called ResultsPage.js and add:
import React, { useState, useEffect } from "react";
import { withRouter } from "react-router-dom";
import { getAllDeals } from "./requests";
import Card from "react-bootstrap/Card";
import "./ResultsPage.css";
function ResultsPage({ match: { params } }) {
  const [deals, setDeals] = useState([]);
  const [name, setName] = useState("");
  const [initialized, setInitialized] = useState(false);
  const getDeals = async () => {
    const id = params.id;
    const response = await getAllDeals(id);
    setDeals(response.data.content);
    setName(response.data.name);
  };
  useEffect(() => {
    if (!initialized) {
      getDeals();
      setInitialized(true);
    }
  });
  return (
    <div className="results-page">
      <h1 className="center">Deals Results: {name}</h1>
      {deals.map((d, i) => {
        return (
          <Card key={i}>
            <Card.Title className="title">{d.dealer}</Card.Title>
            <Card.Body>
              {d.title}
              <br />
              <a className="btn btn-primary" href={d.link}>
                Go
              </a>
            </Card.Body>
          </Card>
        );
      })}
    </div>
  );
}
export default withRouter(ResultsPage);
We get the web scrape job entry by ID and display the results in a list. We get the ID by checking for the ID parameter in the route. When we wrap ResultsPage with the withRouter function, we get the query parameter in the props.
For styling, we create a file called ResultsPage.css and add:
.results-page {
  padding: 20px;
}
.title {
  margin: 0 20px;
}
Now we need to create the MobX store to store the jobs data. Create a file stores.js and add:
import { observable, action, decorate } from "mobx";
class JobsStore {
  jobs = [];
setJobs(jobs) {
    this.jobs = jobs;
  }
}
JobsStore = decorate(JobsStore, {
  jobs: observable,
  setJobs: action,
});
export { JobsStore };
The setJobs is used in HomePage.js to set the jobs, and we accessed the list of jobs stored in the jobs field by using jobsStore.jobs since we passed in an instance of this as a prop to HomePage .
Finally, we create the top bar for our page to display the title and link for the app. Create TopBar.js and add:
import React from "react";
import Navbar from "react-bootstrap/Navbar";
import Nav from "react-bootstrap/Nav";
import NavDropdown from "react-bootstrap/NavDropdown";
import "./TopBar.css";
import { withRouter } from "react-router-dom";
function TopBar({ location }) {
  const { pathname } = location;
  return (
    <Navbar expand="lg" variant="dark">
      <Navbar.Brand href="#home">Web Scraper App</Navbar.Brand>
      <Navbar.Toggle aria-controls="basic-navbar-nav" />
      <Navbar.Collapse id="basic-navbar-nav">
        <Nav className="mr-auto">
          <Nav.Link href="/" active={pathname == "/"}>
            Home
          </Nav.Link>
        </Nav>
      </Navbar.Collapse>
    </Navbar>
  );
}
export default withRouter(TopBar);
This just displays the Navbar provided by React Bootstrap with the link to the home page. If we are on the home page, then the link will be highlighted. This is the purpose of the active property.
Next we create TopBar.css and add:
nav {
    background-color: green!important;
}
Finally, in index.html in the public folder, replace the existing code with:
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="shortcut icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta
      name="description"
      content="Web site created using create-react-app"
    />
    <link rel="apple-touch-icon" href="logo192.png" />
    <!--
      manifest.json provides metadata used when your web app is installed on a
      user's mobile device or desktop. See https://developers.google.com/web/fundamentals/web-app-manifest/
    -->
    <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />
    <!--
      Notice the use of %PUBLIC_URL% in the tags above.
      It will be replaced with the URL of the `public` folder during the build.
      Only files inside the `public` folder can be referenced from the HTML.
Unlike "/favicon.ico" or "favicon.ico", "%PUBLIC_URL%/favicon.ico" will
      work correctly both with client-side routing and a non-root public URL.
      Learn how to configure a non-root public URL by running `npm run build`.
    -->
    <title>Web Scraper App</title>
    <link
      rel="stylesheet"
      href="https://maxcdn.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css"
      integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T"
      crossorigin="anonymous"
    />
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
    <!--
      This HTML file is a template.
      If you open it directly in the browser, you will see an empty page.
You can add webfonts, meta tags, or analytics to this file.
      The build step will place the bundled scripts into the <body> tag.
To begin the development, run `npm start` or `yarn start`.
      To create a production bundle, use `npm run build` or `yarn build`.
    -->
  </body>
</html>
We added the Bootstrap styles and change the title.
Now that the hard work is done. We can run the app.
First run the backend app by running npm start from the backend folder. Next go to the frontend folder and run npm start. When it asks you if you want it to run it from a different port, type yes.
At the end we get:


Level Up Coding
Coding tutorials and news.
Follow
256
JavaScript
Web Scraping
Nodejs
React
Expressjs
256 claps



John Au-Yeung
WRITTEN BY

John Au-Yeung
Follow
Web developer. Subscribe to my email list now at http://jauyeung.net/subscribe/ . Follow me on Twitter at https://twitter.com/AuMayeung
Level Up Coding
Level Up Coding
Follow
Coding tutorials and news. The developer homepage gitconnected.com
Write the first response
More From Medium
5 Mindsets of Unsuccessful Developers
Manish Jain in Level Up Coding

No Cookies, No Problem — Using ETags For User Tracking
Nicolas Hinternesch in Level Up Coding

6 JavaScript Code Snippets For Solving Common Problems
Tate Galbraith in Level Up Coding

6 Mistakes Every Developer has Made
Daan in Level Up Coding

How Do You Determine your Salary As A Startup CEO?
brett fox in Level Up Coding

React’s useImperativeHandle made simple
Dylan Kerler in Level Up Coding

6 Python Packages you should be using in every Django Web App
Ordinary Coders in Level Up Coding

5 JavaScript Strings Tricks You Should Know
Anupam Chugh in Level Up Coding

Discover Medium
Welcome to a place where words matter. On Medium, smart voices and original ideas take center stage - with no ads in sight. Watch
Make Medium yours
Follow all the topics you care about, and we’ll deliver the best stories for you to your homepage and inbox. Explore
Become a member
Get unlimited access to the best stories on Medium — and support writers while you’re at it. Just $5/month. Upgrade
About
Help
Legal