---
layout: post
title: Working with Flask, MongoDB and CharJS for nice data visualization with python
date: 2020-02-20 12:00:00 -0500
description: Sharing some tips to visualize data in python
img: posts_imgs/flask_mongo.jpg 
tags: [python, mongodb, data visualization, flask]
---

In this post i want to share some major tips to visualize data using python and mongodb. For the visualization part we use ChartJS although any other library available might be also suitable. For this post we will use an existing dataset so the main focus will be: Inititalize your flask app, database connection, data query, passing the data to template engine and use the data to generate charts.

## Flask app

Flask is very easy, light and small python framework to build web servers. I enjoy using it a lot because of its flexibility. To start using it (asuming you have already installed the package) you only need following code snippet:

```python
    from flask import Flask, render_template, url_for

    app = Flask(__name__)

    @app.route('/')
    def index():
        return render_template('index.html')

    if __name__ == "__main__":
        app.run(host='0.0.0.0',debug=True)
```

Voilà, now you already have a web server running on your 127.0.0.1:5000

## MongoDB Connection

In my case, my data is saved in a MongoDB database in an AWS EC2 linux instance manually created. So to connect to the database we need to authenticate in the Ec2 instance before. Usually these connections are secured with SSH protocol. So in python we need to install the library *sshtunnel* and use the package *SSHTunnelForwarder*.

To create the connection we need to fill following data:

{% highlight python %}
def connectSSH():
    server = SSHTunnelForwarder(
    (os.getenv("MONGO_HOST"), 5552),
    ssh_username=os.getenv("MONGO_SSH_USER"),
    ssh_private_key=os.getenv("MONGO_SSH_PRIVATE_KEY"),
    remote_bind_address=(os.getenv("MONGO_LOCALHOST"), 27017),
    local_bind_address=(os.getenv("MONGO_LOCALHOST"), 27017),
    )
    return connectionSSH
{% endhighlight %}

MONGO_HOST is the ip address of the EC2 instance
MONGO_SSH_USER is the user of the EC2 instance, depending on the type of AMI used it could be: ec2-user, ubuntu, root, admin etc.
MONGO_SSH_PRIVATE_KEY is the path where your PEM key is located in case you use this method to connect to the instance.
MONGO_LOCALHOST is the address where the MongoDB is installed, if you follow normal procedure this would be the localhost address (127.0.0.1)

The previous code returns *connectionSSH* which shoould be started like this (assuming it is saved in a variable called server):

{% highlight python %}
    server.start()
{% endhighlight %}

Once you finish your work on the instance, you should manually close the connection to avoid problems (the library usually fails when we kept the connection open):

{% highlight python %}
    server.stop()
{% endhighlight %}

So now you have tested the SSH connection we can connect to the Mongo database.

{% highlight python %}
    client = MongoClient(os.getenv("MONGO_LOCALHOST"), 27017)
{% endhighlight %}

At this point you can perform any query on the Mongo database with the variable client.

## MongoDB Query

To perform queries on MongoDB with python we use the library *PyMongo* which is probably the most popular and used library for MongoDB.

To start performing query we need to define the DB where the query will be executed. In the following example we select 'company' as the DB

{% highlight python %}
    db = client['company']
{% endhighlight %}

On the *db* variable we can perform the MongoDB queries, let's do with some examples:

{% highlight python %}
    usersData = db.users.find({"type": 0}, {'_id': 1, 'email': 1, 'created_at': 1})
{% endhighlight %}

Previous query is executed in the collection *users*, the first '{}' is used to add some filters, in this case we are querying all the users where the field *type* is 0. The second '{}' is to define what fields to get from the query, sometimes the collection has too many fields and we only need few, so we use this second array to define which fields to obtain. 

The result are saved as *cursor* object in usersData, that means that you need to iterate over the cursor with a loop similar to this:

{% highlight python %}
    total_users = 0
    recent24h_users = 0
    recentWeek_users = 0
    for item in usersData:
        total_users = total_users + 1
        if item['created_at'] > (datetime.now() - timedelta(hours=24)):      
            recent24h_users = recent24h_users + 1
        if item['created_at'] > (datetime.now() - timedelta(hours=168)): 
            recentWeek_users = recentWeek_users + 1
    
    getKPIUsers = {
            'totalUsers': total_users,
            'recent24hUsers': recent24h_users,
            'recentWeekusers': recentWeek_users
    }
{% endhighlight %}

So in previous code snipet we are counting the total of users registered in the collection, the total users registered in the past 24 hours and 7 days and storing the data in a dict object.

## Passing MongoDB data to HTML templates and graph it

Now that we have our data collected from MongoDB we need to pass it to our view to chart. For this case we use a method provided by flask called *render_template*

{% highlight python %}
    return render_template('dashboard.html', KPIUsers=getKPIUsers)
{% endhighlight %}

Previous code loads the 'dashboard.html' file and send a the data in a variable called KPIUsers.

In the HTML file we have following code to generate the chart:

```html
    {% block head %}
    <!-- Chart.js is loaded here -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.9.3/Chart.min.css">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.9.3/Chart.min.js"></script>
    {% endblock %}
    {% block body %}
    <div>
        <canvas id="users" width="150" height="50"></canvas>
        <script>
        var ctxUsers = document.getElementById('users').getContext('2d');
        var users = new Chart(ctxUsers, {
            type: 'bar',
            data: {
                labels: ['Total', 'Recent 24H', 'Recent Week'],
                datasets: [{
                    label: 'Registro de usuarios',
                    data: [{{KPIUsers['totalUsers']}}, {{KPIUsers['recent24hUsers']}}, {{KPIUsers['recentWeekusers']}}],
                    backgroundColor: 'rgba(63, 191, 127, 0.2)',
                    borderColor: 'rgba(63, 191, 127, 1)',
                    borderWidth: 1
                }]
            }
        });
        </script>
    </div>
    {% endblock %}
```
Notice that the arrangement of the labels and data must be equal to match each KPI with corresponding label. You have tons of options to customize your chart and the [documentation](https://www.chartjs.org/docs/latest/) is very easy to understand.

Hope this post is useful for you, if you have any suggestion/comment/question drop a comment below.

Live long and code! :v::v: