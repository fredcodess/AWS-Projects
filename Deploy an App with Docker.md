#  Compute Project: Containers & AWS Elastic Beanstalk  

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-archi.png?raw=true)


In this project, we will:  
-  Install and use **Docker**  
-  Build a **custom container image**  
-  Run a **containerized application locally**  
-  Deploy the app to **AWS Elastic Beanstalk**, making it live on the web  

---

##  Step 1: Install Docker

1. Head over to the [official Docker website](https://www.docker.com/).
2. Download **Docker Desktop** for your OS.
3. Skip the sign-in step when prompted.
4. Verify installation:

```bash
docker --version
```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-docker-version.png?raw=true)

---

##  Step 2: Run a Pre-Built Container

Run an Nginx container to test things out:

```bash
docker run -d -p 80:80 nginx
```

* `-d` → detached mode (runs in background)
* `-p 80:80` → maps port 80 on your machine to port 80 in the container

Now open [http://localhost](http://localhost) in your browser.

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-nginx-welcome-browser.png?raw=true)

---

##  Step 3: Build The Custom Container

1. Create a project folder:

```bash
cd ~/Desktop
mkdir Compute && cd Compute
touch Dockerfile
```

2. Add the following to your **Dockerfile**:

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

3. Create an **index.html** file:

```html
<!doctype html>
<html>
<head>
  <title>My Web App</title>
</head>
<body>
  <h1>Hello from [Your Name]'s custom Docker image!</h1>
</body>
</html>
```

4. Build The custom image:

```bash
docker build -t my-web-app .
```

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-docker-build-terminal.png?raw=true)

---

##  Step 4: Run The Custom Container

Start the container:

```bash
docker run -d -p 80:80 my-web-app
```

Open [http://localhost](http://localhost) again — you should see your custom HTML page!

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-custom-webpage-browser.png?raw=true)

---

##  Step 5: Deploy to AWS Elastic Beanstalk

1. Go to **AWS Management Console → Elastic Beanstalk**.
2. Create a new application (leave defaults).
3. Choose **Docker** as the platform.
4. Compress the `Dockerfile` and `index.html` into a ZIP file.
5. Upload the ZIP file as application code.
6. Set configuration:

   * **Environment type** → Single instance (Free tier)
   * **Public IP address** → Enabled
   * **Root volume type** → gp3 (10 GB)
   * **System monitoring** → Basic
   * **Managed updates** → Disabled
   * **Deployment policy** → All at once

![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-eb-upload.png?raw=true)
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-compress.png?raw=true)
![Image Alt](https://github.com/fredcodess/AWS-Projects/blob/main/images/dk-aws-eb-live-app.png?raw=true)

---

We have successfully:

* Built and ran a Docker container locally
* Deployed your custom container to AWS Elastic Beanstalk
* Made the app publicly accessible!

