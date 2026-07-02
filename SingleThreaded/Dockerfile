# Dockerfile for the SingleThreaded server variant
# Place this file inside your project's SingleThreaded/ folder before building

FROM eclipse-temurin:17-jdk

WORKDIR /app

# Copy source files from this folder into the image
COPY Server.java .

# Compile
RUN javac Server.java

# NOTE: adjust this if your server's hardcoded port isn't 8010
EXPOSE 8010

CMD ["java", "Server"]
