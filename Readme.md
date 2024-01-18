##Microservice study

- part 1 Unmonolith Attendees: Set up the project 
- part 2 Moving Stuff Around: Moving attendees into its own services, port 8001, attendees in attendees_bc needs to have access to conference VO instanct, updated conference data will be coming from the poller 
- part 3 Syncing Data: poller and synchronize data from the monolith into the attendees microservice

###1. Unmonolith Attendees

![GitHub Logo](/images/UnmonolithAttendees.png)

#####Step-by-step Instructions:

- Fork and clone https://gitlab.com/sjp19-public-resources/microservice-study

- Make data directory with `mkdir data` inside the root of the project directory


```python
mkdir data
```

- Create `events/keys.py` and add the required variables as Learn instructs (Need to add PEXELS_API_KEY and OPEN_WEATHER_API_KEY)

- Run below commands to set up Django and the envirnment 
```python

python -m venv ./.venv
source ./.venv/bin/activate

python -m pip install --upgrade pip

pip install django

pip install requests
pip freeze > requirements.txt

python manage.py makemigrations
python manage.py migrate

```
This project includes a data dump so that you can test out your APIs.

- Run this command to import data:
```python 
python manage.py loaddata conference_go/data.json
```


- Build the image with the following command 
```python
docker build -f [Dockerfile.dev](http://dockerfile.dev/) . -t conference-go-dev
```
- Run the container with the following command 
```python
docker run -it -v "$(pwd):/app" conference-go-dev bash
```
- Inside the shell that that commands runs for you, run the following:
```python
python manage.py migrate
python manage.py loaddata conference_go/data.json
```
- Run the `exit` command inside the shell to exit the docker container
```python 
exit
```

To stop a container, hover over any containers with "conference-go" and "conference-go-dev" in blue letters next to the silly container names and click the trash can.

In your terminal, run these commands to clean up the Docker containers and images:

- Run `docker container prune` to clean up any leftover containers
- Run `docker image prune --all` to clean up any leftover images

Now you can proceed to Step 2: Moving Stuff Around on LearnIf something goes wrong, you can also find the `db.sqlite` file attached to this post. Just download the db file and then put it in your data directory. (edited)

```python
docker build -f [Dockerfile.dev](http://dockerfile.dev/) . -t conference-go-dev

docker run -it -v "$(pwd):/app" conference-go-dev bash

python manage.py migrate

python manage.py loaddata conference_go/data.json

exit

docker container prune

docker image prune --all
```



###2. Moving the monolith

Open a terminal
Open a terminal to the directory that contains the project that you're working in. You're going to use the command line to move things around. It's just a series of commands, of mkdir (make directory) and mv (move) and cp (copy).

Again, make sure you're in the top-level directory of the project. When you type ls and hit Enter, do you see the Django app and project directories, and the manage.py in the list? If not, change your working directory so you're in the right place. Then continue.

These commands move or copy all of the non-attendees stuff into a directory named monolith/:

![GitHub Logo](/images/MovingStuffAround.png)

```python
mkdir monolith
mv accounts monolith
cp -r common monolith
mv conference_go monolith
mv data monolith
mv events monolith
mv presentations monolith
mv Dockerfile monolith
cp Dockerfile.dev monolith
mv manage.py monolith
cp requirements.txt monolith
```

Check to make sure that what you have is what you see in the picture inside the monolith/ directory.

And note that by moving the events directory your keys.py file has been moved and the rule for it in your .gitignore will no longer apply, unless you update it to the new path.

Moving the rest
Now create the directory for the new microservice. Move the rest of the stuff in there.

```python
mkdir attendees_microservice
mkdir attendees_microservice/data
mv attendees attendees_microservice
mv common attendees_microservice
mv Dockerfile.dev attendees_microservice
mv requirements.txt attendees_microservice
```
Check to make sure that what you have is what you see in the picture inside the attendees_microservice/ directory.

Fix up the monolith
Open Visual Studio Code to the project, if you haven't yet. We need to remove the references of the attendees Django app from the monolith. There are two places those occur, in the Django project's settings.py and the urls.py.

We also know that we need to update the ALLOWED_HOSTS to allow the microservice to access it via the hostname "monolith". We'll fix that up, too.

Open monolith/conference_go/settings.py.

Find the entry for attendees in the INSTALLED_APPS list and remove it
Set the ALLOWED_HOSTS to ["localhost", "monolith"]
Open monolith/conference_go/urls.py. Find the line that includes "attendees.api_urls" and delete it.

- Back in your terminal, build a new Docker image for the monolith.

```python
cd monolith

docker build -f monolith/Dockerfile.dev ./monolith -t conference-go-dev
```

- Run the new image and test it in Insomnia.
- need to run this in the top directory

```python
docker run -v "$(pwd)/monolith:/app" -p "8000:8000" conference-go-dev
```
Everything should work except the Attendees urls. They're over in the other microservice now.

Once you're done testing, stop the container with Control+C. If that doesn't work, remember you can use Docker Dashboard to stop the container.


Change the microservice port number
In attendees_microservice/Dockerfile.dev, update the command to use port 8001 instead of 8000.

Create the attendees development image
It's time to use Docker to work with Django. We need to create a development Docker image for the attendees microservice.

```python
cd attendees_microservice

docker build -f attendees_microservice/Dockerfile.dev ./attendees_microservice -t attendees-microservice-dev

```
Now that we have that image built, we can run a container to use the django-admin tool to create a new Django project for the microservice.

```python
docker run -it -v "$(pwd)/attendees_microservice:/app" attendees-microservice-dev bash
```

Once the container is running, you'll know it because your command prompt will change. You can create a new Django project now. **Make sure to type the period (.) at the end of the command.

```python
django-admin startproject attendees_bc .
```

Configure the new project
In Visual Studio
In Visual Studio Code, you should see the new attendees_microservice/attendees_bc directory. That's what contains the Django project for the microservice. We need to change the settings.py and urls.py files in there like we would with any Django project that needs to load one of our Django apps.

Open attendees_microservice/attendees_bc/settings.py and make the following changes:

```python
# In settings.py
INSTALLED_APPS = [
    # ADD THIS LINE
    "attendees.apps.AttendeesConfig",
    ...
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",

    # DELETE OR COMMENT OUT THIS LINE
    # "django.middleware.csrf.CsrfViewMiddleware",

    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

...

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",

        # Change this line by adding data/ in front of the db.sqlite3
        # "NAME": BASE_DIR / "db.sqlite3",
        "NAME": BASE_DIR / "data/db.sqlite3",
    }
}
```

Open
Open attendees_microservice/attendees_bc/urls.py and make the following changes:

```python
# In urls.py
from django.contrib import admin
from django.urls import path, include   # ADD INCLUDE

urlpatterns = [
    path("admin/", admin.site.urls),

    # THIS IS A NEW LINE
    path("api/", include("attendees.api_urls")),
]


```

Add the value object model
In attendees_microservice/attendees/models.py, add the new model ConferenceVO to replace the old Conference model that's in another Bounded Context. The VO suffix reminds us that it's a Value Object.


```python

class ConferenceVO(models.Model):
    import_href = models.CharField(max_length=200, unique=True)
    name = models.CharField(max_length=200)
```
Further down, where the Attendee references "events.Conference", replace that reference with the ConferenceVO class.

Fix the encoder in the view
In attendees_microservice/attendees/api_views.py, we need to get rid of all imports from the events module. Then, add ConferenceVO to the list of things imported from the .models module.

Add this new encoder to the file:

```python
class ConferenceVODetailEncoder(ModelEncoder):
    model = ConferenceVO
    properties = ["name", "import_href"]
```
Everywhere that used ConferenceListEncoder, replace it with the new ConferenceVODetailEncoder.

Fix the list attendees function
We're not dealing with conferences anymore. We're dealing with ConferenceVO objects. Because of that, let's change the name of the parameter of the api_list_attendees function to reflect that.

```python
# UPDATE THE PARAMETER NAME
def api_list_attendees(request, conference_vo_id=None):

```

In the GET handler portion of the function, we used the old conference_id parameter. Update that to the new name.

```python
# UPDATE THE ARGUMENT NAME
attendees = Attendee.objects.filter(conference=conference_vo_id)
```

Finally, down in the part that handles the POST, we need to update the use of Conference to ConferenceVO.

```python 
else:
    content = json.loads(request.body)

    try:
        # THIS LINE IS ADDED
        conference_href = f'/api/conferences/{conference_vo_id}/'

        # THIS LINE CHANGES TO ConferenceVO and import_href
        conference = ConferenceVO.objects.get(import_href=conference_href)

        content["conference"] = conference

           ## THIS CHANGES TO ConferenceVO
    except ConferenceVO.DoesNotExist:
        return JsonResponse(
            {"message": "Invalid conference id"},
            status=400,
        )

    attendee = Attendee.objects.create(**content)
    return JsonResponse(
        attendee,
        encoder=AttendeeDetailEncoder,
        safe=False,
    )
    ```

    Now, the code expects a "conference" key in the submitted JSON-encoded data. It should be the "href" value for a Conference object.

Fix the attendees urls
In attendees_microservice/attendees/api_urls.py, add a new path. Then rename the conferences/<int:conference_id>/ path parameter to match the name of the parameter in the function.


```python
urlpatterns = [
    # ADD THIS LINE
    path("attendees/", api_list_attendees, name="api_create_attendees"),

    path(
        # UPDATE THE PARAMETER NAME
        "conferences/<int:conference_vo_id>/attendees/",
        api_list_attendees,
        name="api_list_attendees",
    ),
    path("attendees/<int:id>/", api_show_attendee, name="api_show_attendee"),
]
```

Delete the old migration
In attendees_microservice/attendees/migrations, delete all of the migrations that begin with numbers. Do not delete the __init__.py file.

Make migrations
Over in your terminal that's running the development Dockerfile for the attendees microservice, make your migrations. Then apply the migrations.

Once those run, let's try running the development Dockerfile for real.

Exit out of the container by typing exit and hitting Enter.

Test that the new microservice is running
Run the image letting the Django development Web server start.

Change your active working directory to the attendees_microservice directory and run this command:

```python
docker run -v "$(pwd):/app" -p 8001:8001 attendees-microservice-dev
```

Use Insomnia to make a GET request of the attendees microservice using the url http://localhost:8001/api/conferences/1/attendees/. (Notice the different port number.)

This should return an empty list of attendees.

Stop the container because you're going to need to rebuild the image.




###3. Syncing Data

Now we're going to synchronize data from the monolith into the attendees microservice.

Add a library to requirements.txt
Add the following line to the attendees_microservice/requirements.txt:

```python
django-crontab==0.7.1
```

We don't have to do anything more with that because the next time we build the image for the attendees microservice, Docker will install that into the image!

Check out the package's documentation  on PyPi.


![GitHub Logo](/images/poller.png)

Write the sync script
Create a new file, attendees_microservice/attendees/poll.py. Put the following into that file. If you copy and paste, make sure you understand what it's doing.

```python
import json
import requests

from .models import ConferenceVO


def get_conferences():
    url = "http://monolith:8000/api/conferences/"
    response = requests.get(url)
    content = json.loads(response.content)
    for conference in content["conferences"]:
        ConferenceVO.objects.update_or_create(
            import_href=conference["href"],
            defaults={"name": conference["name"]},
        )
```

Add the script as a cron job
In the
In the attendees_microservice/attendees_bc/settings.py, add the following code:

```python
INSTALLED_APPS = [
    "django_crontab",
    ....
]

CRONJOBS = [
    ("* * * * *", "attendees.poll.get_conferences"),
]
```
Update the Dockerfile
In the
In the Dockerfile.dev file for the attendees microservice, modify it to look like this:

```python 

FROM python:3

# ADD THESE TWO LINES TO INSTALL CRON
RUN apt-get update
RUN apt-get install cron -y

ENV PYTHONUNBUFFERED 1
WORKDIR /app
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

# ADD THESE TWO LINES TO SET UP ROOT CRONTAB
RUN touch /var/spool/cron/crontabs/root
RUN crontab -l

## ALTER THIS LINE TO ADD CRONTABS AND START SERVICE
CMD python manage.py crontab add && service cron start && python manage.py runserver "0.0.0.0:8001"
```

Use the new Dockerfile
Build the image for the new microservice with its polling feature. (You'll need to figure out what directory to run this from. Use your knowledge of directory and file paths to help you.)

```python 

cd attendees-microservice

docker build -f Dockerfile.dev . -t attendees-microservice-dev

```

Create a new network for the monolith and service
This is what we did earlier. We're creating a bridge network for the two services to use to communicate.

```python
docker network create --driver bridge conference-go
```

Test it all out
Start the two
Start the two containers with their names and networks.

This one starts the monolith: (run in the top leverl directory)

```python
docker run -d -v "$(pwd)/monolith:/app" -p "8000:8000" --network conference-go --name monolith conference-go-dev
```
This one starts the microservice: (run in the top leverl directory)
```python
docker run -d -v "$(pwd)/attendees_microservice:/app" -p "8001:8001" --network conference-go --name attendees-microservice attendees-microservice-dev
```
To make sure the polling is happening, run the following command to watch the logs of the monolith. It'll take at most one minute for something to appear.

- (run in the top leverl directory)
```python
docker container logs --follow monolith
```
Type Control+C to exit out of that.

If no logs appear, look at the logs for the microservice.

- (run in the top leverl directory)
```python 
docker container logs --follow attendees-microservice
```
Clean up
```python

docker container stop attendees-microservice monolith
docker container rm attendees-microservice monolith
```
