---
layout: post
title: "How do I make my data available to the client scripts?"
date: 2017-10-09 10:52:54
image: '/assets/img/'
description:
tags: howto project
categories:
---

As we have seen during the class, the architecture of our GitHub Analytics project was designed to deal with a couple of constraints. In our case, the following constraints are key: ease of use, ease of maintenance and cost. 

Heroku is a Platform-as-a-Service (PaaS) provider, which gives us free credits to deploy services. If we use these credits wisely, then we can deploy our project at no cost and it will stay online forever, without the need for us to do anything. Cool.

## Initial idea (fail)

Our initial idea was to split the system in 3 parts:

- The agent (or crawler): it is responsible to extract data from GitHub, via the GitHub REST API. It is a task that is executed periodically (e.g. daily or hourly). When it is executed, it most likely will only run for a couple of minutes. Thanks to the Heroku billing system, this means that we will only pay a few credits every time we generate data.
- The font-end assets: this are the HTML, CSS and JS files that are sent to the browser and that provide the UI to the system. We have seen that using GitHub Pages is a an easy way to publish these assets, at no cost and without any maintenance effort.
- The back-end server: it was supposed to be responsible to store the data generated by the agent. In the front-end, at some point our script is going to make an AJAX request to some server in order to get the data. It cannot contact the agent directly (because it is only awake on a daily or hourly basis). So, the idea was to create a simple app with the express.js framework. The agent could send a `POST` HTTP request to upload data. The font-end script could send a `GET` request to download it. We were planning to deploy the back-end on Heroku as well.

## Why the initial idea failed

The problem with this approach is that Heroku does not provide us with file-system persistence. When we develop and application for Heroku, we can use the file system APIs and store temporary files. However, if Heroku has to shutdown and restart our app, then our file system will be flushed.

In addition, because of the cost constraint, we use Heroku in such a way that if no user is accessing the app for a while, then it will be automatically paused until someone comes back. Here again, the file system will be flushed.

Fail.

## A better and simpler solution

We considered a few alternatives. One of them was to use a MongoDB add-on available on Heroku. The problem is that this would require additional time to become familiar with the NoSQL database (it is planned for the semester, but right now is not a good idea). Another alternative was to use the S3 storage API provided by Amazon Web Services. But here again, this would require time to learn and to configure another free environment.

Since we already use GitHub Pages, it would actually be interesting to store the data file together with the other front-end assets. In other words, in the same GitHub repo, we would have HTML, CSS, JS files as well as the data generated periodically by the agent.

But how can we update this file? We don't want to upload it manually, so how can we automate this process?

The answer is to use the GitHub API for that purpose. In the project, we use the GitHub API to read data (statistics, commits, pull requests, etc.). But in fact, we can also write data with the API. As a matter of fact, we can even create and push commits with the API. This means that when the agent has generated a new version of the file, it should be able to push a new commit to our front-end repo.

## From theory to practice

To implement this idea, we could use [these endpoints](https://developer.github.com/v3/git/) of the public GitHub API.

But other people have already had similar ideas and have already done the hard work. One example is the [`github-publish`](https://github.com/voxpelli/node-github-publish) npm module. It gives us a function to upload a file to a GitHub repo (there is one option that allows us to override the file if it exists).

So, let't implement a `Storage` class that our agent can use whenever it wants to upload a new version of the file. Here is the behavior of the file, expressed as a mocha test:

{% highlight javascript %}
const Storage = require('../src/storage');
const { token } = require('../github-credentials.json');
const should = require('chai').should();

describe('Storage', () => {
  it('should allow me to store a file on GitHub', (done) => {
    const repo = 'Teaching-HEIGVD-TWEB-GitHubAnalytics-Frontend';
    const username = 'SoftEng-HEIGVD';
    const storage = new Storage(username, token, repo);
    const content = {
      random: Math.random(),
    };
    storage.publish('my-data-file.json', JSON.stringify(content), 'new version of the file', (err, result) => {
      should.not.exist(err);
      should.exist(result);
      done();
    });
  });
});
{% endhighlight %}

The file `../github-credentials` contains a personal access token created via the GitHub Web UI. You will of course have to replace `SoftEng-HEIGVD` and `Teaching-HEIGVD-TWEB-GitHubAnalytics-Frontend` with your user and repo names.

Let's now have a look at the implementation of the `Storage` class:

{% highlight javascript %}
const GitHubPublisher = require('github-publish');

class Storage {
  constructor(username, token, repo) {
    this.username = username;
    this.token = token;
    this.repo = repo;
    this.publisher = new GitHubPublisher(token, username, repo);
  }

  publish(path, content, commitMessage, done) {
    const options = {
      force: true,
      message: commitMessage,
    };
    this.publisher.publish(path, content, options)
      .then((result) => {
        done(undefined, result);
      })
      .catch((err) => {
        done(err);
      });
  }
}

module.exports = Storage;
{% endhighlight %}

As a matter of fact, our `Storage` class does not do much on top of the `github-publish` module, so we might use it directly as well.

One thing that we have done, because we have not seen `Promises` in the course yet, is to make the result available through a callback.



