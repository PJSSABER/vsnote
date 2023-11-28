# config in remote
- viewing the temp file fish creates to see what port the webserver is bound to (I had 8000)
- opening a second ssh session and tunneling the port to localhost. example:
    ssh -L 8000:localhost:8000 user@host
- opening my browser to the link referenced in the temp file.(most time fish will return that for you)
    example outpu: Web config started at file:///tmp/web_configolx2xosi.html