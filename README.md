# MERN Stack Application Deployment Troubleshooting and Resolution Report

---

**Prepared by:** GitHub Copilot Assistant  
**Date:** April 18, 2026  
**Project:** Travel Memory MERN Application  
**Infrastructure:** AWS EC2 (Ubuntu 22.04)  
**Tools:** Ansible, PM2, Nginx, MongoDB  

---

## Executive Summary

This report details the systematic troubleshooting and resolution of deployment issues encountered while automating the deployment of a MERN stack application using Ansible. The application, Travel Memory, allows users to record and share travel experiences. Through iterative debugging and configuration fixes, all critical issues were resolved, resulting in a fully functional deployed application.

## Project Overview
This report documents the troubleshooting and resolution of issues in an Ansible playbook designed to deploy a MERN (MongoDB, Express.js, React, Node.js) stack application on AWS EC2 instances. The application is a travel memory app with frontend, backend, and database components.

## Initial Setup
- **Infrastructure**: Two Ubuntu EC2 instances (web server and database server)
- **Tools**: Ansible for automation, PM2 for process management, Nginx as reverse proxy
- **Source**: GitHub repository (TravelMemory app)

## Issues Encountered and Solutions

### 1. Node.js Installation Failure
**Problem**: Ubuntu default Node.js package was outdated and incompatible with the app requirements.

**Solution**: 
- Switched to NodeSource repository for Node.js 18
- Added `curl -fsSL https://deb.nodesource.com/setup_18.x | bash -` before apt install

### 2. Git Repository Clone Conflicts
**Problem**: Ansible git module failed when local modifications existed in the cloned directory.

**Solution**:
- Added `force: yes` to the git task to overwrite any existing changes

### 3. Frontend Build Hanging
**Problem**: React build process would hang indefinitely during Ansible execution.

**Solution**:
- Made the build task asynchronous with `async: 3600` and `poll: 30`
- Set environment variables: `CI=false`, `GENERATE_SOURCEMAP=false`, `NODE_OPTIONS=--max-old-space-size=4096`
- Logged build output to `/tmp/frontend-build.log` for debugging

### 4. Backend Startup Issues
**Problem**: PM2 was trying to start `server.js` which didn't exist; actual entry point was `index.js`.

**Solution**:
- Changed PM2 command to `pm2 start index.js --name travel-backend -f`

### 5. Nginx 500 Internal Server Errors
**Problem**: Static React build files were inaccessible due to permission issues.

**Solution**:
- Added permission fixes: `chown -R ubuntu:www-data /home/ubuntu/TravelMemory/frontend/build`
- Set execute permissions on parent directories: `chmod o+X /home/ubuntu /home/ubuntu/TravelMemory /home/ubuntu/TravelMemory/frontend`

### 6. Database Connection Failures
**Problem**: MongoDB was binding only to localhost (127.0.0.1), preventing remote connections from the web server.

**Solution**:
- Modified `/etc/mongod.conf` to set `bindIp: 0.0.0.0`
- Created database user in the correct database (`travelmemory` instead of `admin`)
- Reordered Ansible playbook to configure database **before** starting the web server

### 7. Frontend API Routing Issues
**Problem**: React app was making API calls to `http://localhost:3001` but Nginx was proxying to local backend.

**Solution**:
- Set `REACT_APP_BACKEND_URL=/trip` during build
- Added Nginx location block for `/trip` proxying to `127.0.0.1:3000`

## Configuration Changes Made

### Ansible Playbook (site.yml)
- Reordered execution: database setup → web server setup → security hardening
- Added MongoDB configuration tasks
- Updated Node.js installation method
- Enhanced frontend build process
- Fixed backend startup command
- Added Nginx proxy configuration for `/trip` endpoint

### Nginx Configuration
```
location / {
    root /home/ubuntu/TravelMemory/frontend/build;
    index index.html;
    try_files $uri /index.html;
}

location /trip {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

### MongoDB Configuration
- bindIp changed from 127.0.0.1 to 0.0.0.0
- User created with readWrite access on travelmemory database

## Final Status
The MERN application is now successfully deployed and accessible. The frontend loads without JavaScript errors, and API calls to retrieve and add travel experiences work correctly.

## Screenshots
[To be added by user]

## Lessons Learned
- Always configure databases before starting application servers in deployment scripts
- Use asynchronous tasks for long-running build processes in Ansible
- Ensure proper file permissions for web server access
- Test API endpoints after deployment to verify connectivity
- Use environment variables to configure frontend API URLs for different deployment environments

## Tools Used
- Ansible for infrastructure automation
- PM2 for Node.js process management
- Nginx for reverse proxy and static file serving
- MongoDB for data persistence
- React for frontend UI
- Express.js for backend API