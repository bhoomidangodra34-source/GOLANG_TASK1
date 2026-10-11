## First, understand the basic setup

Before writing the handler functions, we need a place to store our tasks.

```go
type Task struct {
    ID    string `json:"id"`
    Title string `json:"title"`
}

var tasks = []Task{
    {ID: "1", Title: "Learn Go"},
    {ID: "2", Title: "Build REST API"},
}
```

- `type Task struct` defines what a task contains.
- `ID` uniquely identifies a task.
- `Title` stores its name.
- `json:"id"` and `json:"title"` tell Go what field names to use when converting the struct to JSON.
- `var tasks = []Task{...}` creates a slice containing two sample tasks.

For now, this slice acts as our database. The tasks will be stored in memory, so they won't persist after the server restarts.

---

## 1. GET — Read all tasks

**GET /tasks**

```go
func getTasks(c *gin.Context) {
    c.IndentedJSON(http.StatusOK, tasks)
}
```

Let's break it down.

- `func getTasks(...)` defines the handler function for retrieving tasks.
- `c *gin.Context` gives Gin access to the incoming request and the response we send back.
- `c.IndentedJSON(...)` converts the data into formatted JSON and sends it to the client.
- `http.StatusOK` means HTTP status code `200`, indicating success.
- `tasks` is the slice we want to return.

**Expected response:**

```json
[
  {
    "id": "1",
    "title": "Learn Go"
  },
  {
    "id": "2",
    "title": "Build REST API"
  }
]
```

In Q1.1, you printed `"GET /tasks"`. Now, instead of printing a message, you return actual task data.

## 2. POST — Create a task

**POST /tasks**

```go
func createTask(c *gin.Context) {
    var newTask Task

    if err := c.ShouldBindJSON(&newTask); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid request"})
        return
    }

    tasks = append(tasks, newTask)
    c.JSON(http.StatusCreated, newTask)
}
```

**Logic, step by step:**

1. `var newTask Task` creates an empty task variable to hold the incoming data.
2. `c.ShouldBindJSON(&newTask)` reads the JSON body sent by the client and puts its values into `newTask`.
3. If the JSON cannot be parsed, we return `400 Bad Request`.
4. `append(tasks, newTask)` adds the new task to our slice.
5. `c.JSON(http.StatusCreated, newTask)` returns the created task with status `201 Created`.

For example, send this JSON in Postman:

```json
{
  "id": "3",
  "title": "Learn Gin"
}
```

The new task is added to the slice.

**Why `&newTask`?** The `&` passes a pointer, allowing Gin to populate the actual variable with the incoming JSON values.

---

## 3. PUT — Update a task

**PUT /tasks/:id**

```go
func updateTask(c *gin.Context) {
    id := c.Param("id")

    var updatedTask Task
    if err := c.ShouldBindJSON(&updatedTask); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid request"})
        return
    }

    for i, task := range tasks {
        if task.ID == id {
            updatedTask.ID = id
            tasks[i] = updatedTask
            c.JSON(http.StatusOK, updatedTask)
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "Task not found"})
}
```

**Logic, step by step:**

1. `c.Param("id")` reads the task ID from the URL.
2. `var updatedTask Task` creates a variable for the new task data.
3. `ShouldBindJSON` reads the replacement data from the request body.
4. `for i, task := range tasks` loops through every task. `i` is its index, and `task` is its current value.
5. `if task.ID == id` checks whether the current task matches the requested ID.
6. `updatedTask.ID = id` keeps the ID from the URL.
7. `tasks[i] = updatedTask` replaces the existing task at that index.
8. If the task isn't found, the server returns `404 Not Found`.

Example URL: `PUT /tasks/2`

Example body:

```json
{
  "title": "Learn Go and Gin"
}
```

This simple example expects the client to send the replacement task data. For a more robust API, we would also validate the title and require it to be non-empty.

---

## 4. DELETE — Delete a task

**DELETE /tasks/:id**

```go
func deleteTask(c *gin.Context) {
    id := c.Param("id")

    for i, task := range tasks {
        if task.ID == id {
            tasks = append(tasks[:i], tasks[i+1:]...)
            c.Status(http.StatusNoContent)
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "Task not found"})
}
```

**Logic, step by step:**

1. `c.Param("id")` extracts the task ID from the URL.
2. The `for` loop searches the slice for that task.
3. `if task.ID == id` checks whether we found it.
4. `tasks[:i]` selects the elements before the task.
5. `tasks[i+1:]` selects the elements after it.
6. `append(tasks[:i], tasks[i+1:]...)` combines those two parts, leaving out the task we want to remove.
7. `c.Status(http.StatusNoContent)` returns status `204`, meaning the deletion succeeded and there is no response body.
8. If no matching task exists, the function returns `404 Not Found`.

Example URL: `DELETE /tasks/2`

---

## How all four functions fit together

| Method | Handler | What it does | Success status |
|---|---|---|---|
| GET | `getTasks` | Returns all tasks | `200 OK` |
| POST | `createTask` | Adds a task | `201 Created` |
| PUT | `updateTask` | Replaces a task | `200 OK` |
| DELETE | `deleteTask` | Removes a task | `204 No Content` |

Your routes in `main()` should point to these same functions:

```go
router := gin.Default()

router.GET("/tasks", getTasks)
router.POST("/tasks", createTask)
router.PUT("/tasks/:id", updateTask)
router.DELETE("/tasks/:id", deleteTask)

router.Run(":8080")
```
This is the basic CRUD implementation for learning. It uses in-memory storage, accepts client-provided IDs.