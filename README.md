HTTP server in C. No frameworks, No libraries
---

Compile:

```bash
gcc -o server server.c
./server
```

You should see:

```
Server running on http://localhost:8080
```

Now open your browser and go to:

- `http://localhost:8080/` - You should see "Hello from my C server!"
- `http://localhost:8080/about` - Should show the about page
- `http://localhost:8080/anything-else` - Should return a 404

You can also test with `curl`:

```bash
curl -v http://localhost:8080/
```
