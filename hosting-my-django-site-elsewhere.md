Hosting my Django Site elsewhere

If you build your website using [pedestrian.site](https://pedestrian.site/) -- then one of the
benefits of the extremely simple architecture your application has, is that you can move it 
elsehwere extremely easy. It is supposed to be a self-contained Python Django application with
very little requirements on infrastructure. Now, obviously, if you wish to run your application
for many, many thousands of users -- these architectural choices might need a revisit; fixing
that is not too hard though :-)

# Step 1, Create a webapplication

I created an application with this prompt:

> Create an application where users can register, login -- then they should put in their home address, 
> please do an openstreetmap preview of the location - once done, they should be able to register dogs
> passing by their house.

It can be seen here: [e1e82234-ce2b-4b8b-9a4c-d6312a0ad007.pedestrian.site/](https://e1e82234-ce2b-4b8b-9a4c-d6312a0ad007.pedestrian.site/).
It's really simple. When you create your own, do test it online, and tell Mistral AI to fix whatever is
not working or just wrong.

# Step 2, Downloading..

If you log in to pedestrian.site and 'view details' on your application, you will see a button
called 'Download App'. If you click that one, you will receive a zip-file containing your
application. Do that. I did that with my application; please follow these steps to run it from
your laptop (example should work for Mac and Linux):

    $ unzip -q Dogs_at_my_House.zip
    $ cd Dogs_at_my_House
    $ python3 -m venv venv
    $ source venv/bin/activate
    (venv) $ pip3 install -r requirements.txt

At this point, you should edit the `settings.py` file located in the project subdirectory, in my case `dogwatch/settings.py` - the following
settings need to change:

    HOSTNAME
    LOGGING / handlers / file / filename

Eventually you might consider flipping 'DEBUG' to 'False', as running it with 'True' can expose details about your application
that you would not want to expose. With this, you can run the application by doing:

    (venv) $ python3 manage.py runserver

# Step 3, Further Considerations

Now, you might want to change certain things to harden this application and provide a more solid infrastructure, this could be:

- Using a reverse proxy to handle certificates and thus TLS traffic
- Installing a relational database and not use SQLite
- Have it run by systemd/launchd
- Fronting it with a CDN

All this is not hard to do; the Django framework and the community around Django provides plenty of documentation. Your case might
differ ever so slightly from the tutorials, so do not be afraid to reach out.


Tags: python, django, software
