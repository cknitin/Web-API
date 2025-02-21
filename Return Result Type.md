# Return Result Type

What are the type of return result in dot net core webapi

In .NET Core Web API, there are several types of return results you can use to respond to HTTP requests. Here's an overview of some common IActionResult implementations:

## 1. OkResult

Status Code: 200 OK
Use Case: When the request was successful, and you want to return either no content or content with the success status.

```
return Ok(); // No content
return Ok(someObject); // With content
```
## 2. CreatedResult

 Status Code: 201 Created
 Use Case: When a new resource is created, typically used with POST requests. It can include a location header pointing to the new resource.

```
return CreatedAtAction(nameof(GetById), new { id = newObject.Id }, newObject);
```

## 3. BadRequestResult

Status Code: 400 Bad Request
Use Case: When the request cannot be processed due to client errors, like invalid input.

```
return BadRequest();
return BadRequest("Invalid input"); // With a message
```

## 4. NotFoundResult

Status Code: 404 Not Found
Use Case: When the requested resource is not found.

csharp
return NotFound();

## 5. NoContentResult

Status Code: 204 No Content
Use Case: When the request was successful, but there is no additional content to return.

```
return NoContent();
```

## 6. UnauthorizedResult

Status Code: 401 Unauthorized
Use Case: When authentication is required and has failed or has not been provided.

```
return Unauthorized();
```

7. ForbiddenResult

Status Code: 403 Forbidden
Use Case: When the server understood the request but refuses to authorize it.

```
return Forbid();
```

## 8. ConflictResult

Status Code: 409 Conflict
Use Case: When the request could not be processed because of a conflict in the current state of the resource.

```
return Conflict();
```

## 9. StatusCodeResult

Status Code: Any HTTP status code you specify.
Use Case: When you want to return a specific HTTP status code not covered by the other results.

```
return StatusCode(418); // HTTP 418 I'm a teapot
```

## 10. ObjectResult

Status Code: Variable, depends on how it's used.
Use Case: A more generic result type that can be used to return any object with an associated status code.

```
return new ObjectResult(someObject) { StatusCode = 200 };
```

## 11. JsonResult

    Status Code: Typically 200 OK, but can be set.
    Use Case: When you want to return JSON formatted data explicitly.

csharp
return new JsonResult(someObject);

## 12. ContentResult

Status Code: Can be set, typically 200 OK.
Use Case: When you need to return plain text or other content with specific content type.

```
return Content("Some plain text", "text/plain");
```

## 13. RedirectResult

Status Code: 302 Found by default, can be changed.
Use Case: When you want to redirect the client to another URL.

```
return Redirect("~/someController");
```

These return types can be used either directly or through helper methods like Ok(), BadRequest(), etc., which are extension methods on the ControllerBase class. They provide a standardized way to communicate the outcome of an API call to the client. Remember, you can also return custom results by implementing IActionResult if none of the predefined result types meet your specific needs.

