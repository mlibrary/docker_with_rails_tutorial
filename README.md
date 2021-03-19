Assignments:
Use Docker Compose to set up an app with:
* Rails
* Mariadb
* Webpack Dev Server

Steps
1. cd to exercise diretory
2. Look at the files:
   * Gemfile. (it only has rails in it)
   * Gemfile.lock is empty
   * Dockerfile (Go through line by line. Make sure every line is understood.)
   * docker-compose.yml (minimal set up) Has volumes set up for /gems to speed things up
   * env/database with some environment variables
3. `docker-compose build web`
4. `docker-compose run --rm web bash` and look around. Is there vim? What folder are you in? Who are you? Are there really gems in /gems? Create and save a text file in /app with `vim` or just `touch` a file. `ls` in the /app directory to confirm that the new file is there. `exit` the container.
5. `ls` in the directory on the host machine. Is your file there? (The answer will be No, it's not there.) In order for docker to be useful for software development, the container environment needs to be linked to the host environment. This is where volumes comes in.
6. Uncomment lines 8 and 9 in `docker-compose.yml` Then do step 4 again--creating a file in the container and exiting-- and then look for the file on the host machine. It should be there now! This is an example of a "bind mount". The current directory of the host machine is bound to the /app directory of the container. 
7. We need to install rails. `docker-compose run --rm web bundle exec rails new . -d=mysql -T --skip-turbolinks --skip-spring --skip-coffee` (It's fine to overwrite the gemfile) the rails files should be on the host machine now.
8. Go back into the container. `docker-compose run --rm web bash` Look in /gems/ruby/2.7.0/gems. Notice how the mysql gem is missing. This is not ok!. We could rebuild the container doing `docker-compose build web` but that would mean bundle installing everything every time the Gemfile is updated. Instead we'll create a docker volume that can be used to store these gems in development.
9. Uncomment lines 10,30 and 31. This is creating a volume binding between the docker volume "gem_cache" and the folder /gems on the container. This will volume will persist between container instances. Save. Then run `docker-compose run --rm web bundle install` to install the gems in the volume. Then go back into the container `docker-compose run --rm web bash` and check what's in /gems/ruby/2.7.0/gems. Notice all of the gems we bundle installed are there now. We can bundle install new gems as needed. 
10. Next we want to run the basic rails site. In `docker-compose.yml` uncomment lines 6 and 7. This is connecting port 3000 on the contianer to port 3000 on localhost. Save. Then run `docker-compose up`. Then in the browser go to http://localhost:3000
11. Well... almost. `ctrl z` to quit `docker-compose up`. We need to connect the database first. To do that we'll need some environment variables, which already exist in the env/database file. In docker-compose.yml Uncomment lines 11 and 12. If you look in the file env/database, you'll see there are two Environment Variables. MYSQL_ROOT_PASSWORD is an environment variable specific to the mariadb container (see step 13). DATABASE_HOST is 'database' because the services defined in docker-compose.yml can talk to each other by calling their service names. 
12. Edit ./config/database.yml to have the following:
```ruby
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password: <%= ENV.fetch("MYSQL_ROOT_PASSWORD")%>
  host: <%= ENV.fetch("DATABASE_HOST") %>
```
13. Now we actually need mysql, or in this case mariadb. Uncomment lines 14 - 19 and line 32. Line 15: we're pulling the latest mariadb image from dockerhub. It's a container that comes with eveyrthing one needs to run mariadb and not much more. Line 16/17, we're linking the same environment variable file as we did in the web service. Line 18/19 we're creating another docker volume "db_data" to persist the database. Line 25 we're creating another database.   
14. Then we need to create the database. The database needs to be running for this to work. We can put `docker-compose up` in the background or "detached" by adding a `-d` flag. So try `docker-compose up -d database`. Then to see that it's running try `docker-compose ps` to show everything that is or isn't up.
Now run `docker-compose run --rm web bundle exec rails db:create`
15. `docker-compose -up` again. (Add the `-d` flag if you want) This time you'll see that both the web service and the database service are running. Go to http://localhost.com:3000. Huzzah! We are on rails.
16. Lets add a react front end for more complexity! We've already set up webpacker, so now lets put in react: `docker-compose run --rm web bundle exec rails webpacker:install:react`
17. Generate a page to put the react in. `docker-compose run --rm web bundle exec rails generate controller Home index`
Then in app/views/home/index.html.erb put `<%= javascript_pack_tag 'hello_react' %>`
18. Then `docker-compose up`. Then in the browser look at localhost:3000/home/index It works, but it's pretty slow too load.
19. So say we want the webpack-dev-server to do hot reloading. We'll follow the "one process per container rule, and create a separate service that is the same as the the "web" container but has some webpack-dev-server specific changes. In docker-compose.yml uncomment lines 21 - 28. In line 22 we're building from the same Dockerfile as the webservice. In line 23/24 we're opening up 3035 for the webpack dev server. In lines 25/26 we're bind mounting the current directory on the host machine to /app on the container. On line 27/28 we're saying that instead of the command at the end of the Dockerfile (which runs rails) we want to run the bin/webpack-dev-server process instead.
20. Update config/webpacker.yml to include the following starting around line 56:
```ruby
  dev_server:
    https: false
    host: webpacker
    port: 3035
    public: 0.0.0.0:3035
    hmr: true
```
Like with the host for mysql, the host for the webpacker dev server will be the service name. 
21. `docker-compose up` again. The webpacker service will have to build. Ordinarily it shouldn't have to bundle install, but it might this time because we haven't rebuilt since step 3. Go to localhost:3000/home/index in the browser. 
22. Edit the file app/javascripts/packs/hello_react.jsx In line 23 Change "React" to something else. Save and then see that localhost:3000/home/index reloaded with the new text.

