Docker Compose
--------------
Is a service developed in Python.
It is used in the microservice architecure projects.

If we have multiple containers , for example
Order Service
Payment Service
Gateway Service

we can pick the one which will be exposed to outside and the ones that communitcates internally.

we can use networks/volumes inside that docker-compose.yml file.

you do not have to manually create those docker containers.




Install docker-compose:


brew install docker-compose
Create the docker-compose.yaml file as follows:


services:
  auth:
    restart: always
    build: ./auth
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: my-secret-pw
      DB_NAME: authdb
    networks:
      - app-net
  ui:
    restart: always
    build: ./UI
    depends_on:
      - auth
      - weather
    environment:
      AUTH_HOST: auth
      AUTH_PORT: 8080
      WEATHER_HOST: weather
      WEATHER_PORT: 5000
    ports:
      - "3000:3000"
    networks:
      - app-net
  weather:
    restart: always
    build: ./weather
    environment:
      APIKEY: 6d92c50bdamsh81137f3b87ace1fp1d53eejsnfe818b9dbc83
    networks:
      - app-net
  db:
    restart: always
    image: mysql:8.0.25
    environment:
      MYSQL_ROOT_PASSWORD: my-secret-pw
      MYSQL_DATABASE: authdb
    networks:
      - app-net
    volumes:
      - ./db-data:/var/lib/mysql

networks:
  app-net:
    driver: bridge
volumes:
  db-data:
Run the file:


docker-compose up -d
Show the status


docker-compose ps
Bring the application down:


docker-compose down