# NoSQL Brainiac

- [Part 1 - Branch: `first-project-3`](#part-1---branch-first-project-3)
  - [Introduction](#introduction)
  - [Mongoose: Static vs Method](#mongoose-static-vs-method)
  - [Global Error Handler and Unhandled Routes](#global-error-handler-and-unhandled-routes)
- [Part 2 - Branch: `first-project-4`](#part-2---branch-first-project-4)
  - [Higher Order Function](#higher-order-function)
  - [Refactoring Zod validation](#refactoring-zod-validation)
  - [Utils vs Middlewares](#utils-vs-middlewares)
- [Part 3 - Branch: `first-project-5`](#part-3---branch-first-project-5)
  - [Global Error and Not Found Handler (Simplified Example)](#global-error-and-not-found-handler-simplified-example)

[Requiremnet-Analysis](https://docs.google.com/document/d/10mkjS8boCQzW4xpsESyzwCCLJcM3hvLghyD_TeXPBx0/edit?usp=sharing)

[ER Diagram: 1](./ER_Diagram.png)

[ER Diagram: 2](./ER%20Diagram2.png)

# Part 1 - Branch: `first-project-3`

## Table of Contents

- [Introduction](#introduction)
- [Mongoose: Static vs Method](#mongoose-static-vs-method)
- [Global Error Handler and Unhandled Routes](#global-error-handler-and-unhandled-routes)

## Introduction

**SQL:** Sequential Query Language (Oracle, MySQL). Collection, Document, Field

**NoSQL:** No Sequential Query Language (mogoDB, mariaDB, Radis, DynamoDB) Table, Row, Column(Field)

## Mongoose: Static vs Method

In Mongoose, **statics** and **methods** serve different purposes despite both being used to define reusable functions for schemas. The main difference lies in **how** they are used and the **context** in which they operate.

| **Feature**  | **Statics**                                                                                         | **Methods**                                                                                 |
| ------------ | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Context**  | Operates on the **model/class level** (e.g., `Student`).                                            | Operates on the **instance/document level** (e.g., a specific student).                     |
| **Use Case** | For operations that do not require a specific document (e.g., queries, aggregations, or utilities). | For operations related to a specific document (e.g., modifying a property, checking state). |
| **Access**   | Accessed via the **model** (e.g., `Student.findByAge(age)`).                                        | Accessed via the **instance** (e.g., `studentInstance.isAdult()`).                          |

#### **When to Use** `Statics`

Use `statics` when the operation involves the **entire collection** or the model as a whole, and does not pertain to a specific document.

#### Example: Static Method for Finding Documents by Age

```javascript
studentSchema.statics.findByAge = async function (age) {
  return await this.find({ age });
};

// Usage:
const students = await Student.findByAge(18);
```

**When to Use** `Methods`

Use `methods` when the operation involves **an individual document** or needs to modify/work with specific document fields.

#### Example: Method to Check if a Student is an Adult

```javascript
studentSchema.methods.isAdult = function () {
  return this.age >= 18;
};

// Usage:
const student = await Student.findOne({ name: 'John' });
console.log(student.isAdult()); // true or false
```

#### **Key Decision Criteria**

1.  **Does the function involve one document or many?**
    - **One Document:** Use a `method`.
    - **Multiple Documents or the Model Itself:** Use `statics`.
2.  **Do you need access to instance properties (**`**this**`**) like** `**this.age**` **or** `**this.name**`**?**
    - **Yes:** Use a `method`.
    - **No:** Use `statics`.
3.  **Is the operation generic to the model or specific to an instance?**
    - **Generic:** Use `statics`.
    - **Specific to an Instance:** Use `methods`.

## Global Error Handler and Unhandled Routes

```javascript
// Catch-all for unhandled routes
app.use((req, res, next) => {
  const error = new AppError('Route not found', 404);
  next(error);
});

// Global Error Handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({
    error: {
      message: err.message || 'Internal Server Error',
    },
  });
});
```

# Part 2 - Branch: `first-project-4`

## Table of Contents

- [Higher Order Function](#higher-order-function)
- [Refactoring Zod validation](#refactoring-zod-validation)
- [Utils vs Middlewares](#utils-vs-middlewares)

## Higher Order Function

```javascript
import { NextFunction, Request, RequestHandler, Response } from 'express';

const catchAsync = (fn: RequestHandler) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch((err) => next(err));
  };
};

export default catchAsync ;
```

The above function `catchAsync` is a higher order function that takes a function as parameter `fn` and returns `fn` if the `fn` returns the Promise or returns the error.

Now we are passing a async request handler as the param of `catchAsync` function

```javascript
const getSingleStudent = catchAsync(async (req, res) => {
  const { studentId } = req.params;
  const result = await StudentServices.getSingleStudentFromDB(studentId);

  sendResponse(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Student is retrieved succesfully',
    data: result,
  });
});

export const StudentControllers = {
  getAllStudents,
  getSingleStudent,
  deleteStudent,
};
```

The `getSingleStudent` is used in the below middleware where the returned function from `catchAsync` is invoked:

```javascript
router.get('/:semesterIdId', StudentControllers.getSingleStudent);
```

## Refactoring Zod validation

In general we use the zoi validation like:

```javascript
import { StudentServices } from './student.service';
import studentValidationSchema from './student.validation';

const createStudent = async (req: Request, res: Response) => {
  try {
    const studentData = req.body;
    const zodParsedData = studentValidationSchema.parse(studentData);
    const result = await StudentServices.createStudentIntoDB(zodParsedData);
	/...
  } catch (err: any) {
	/...
  }
}
```

We aim to refactor the the `Zod` validation with a middleware, so that we can invoke a single middleware function instead of writing the Schema.Parse multiple times.

First create the `Zod` validation Schema

**student.validation.ts**

```javascript
export const createStudentValidationSchema = z.object({
  body: z.object({
    password: z.string().max(20),
    student: z.object({
      name: userNameValidationSchema,
      gender: z.enum(['male', 'female', 'other']),
      //....
      guardian: guardianValidationSchema,
      localGuardian: localGuardianValidationSchema,
      //....
    }),
  }),
});
```

Next we have to write the middleware function.

**middlewares\\validateRequest.ts**

```javascript
import { NextFunction, Request, Response } from 'express';
import { AnyZodObject } from 'zod';
const validateRequest = (schema: AnyZodObject) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      // validation check
      //if everything allright next() ->
      await schema.parseAsync({
        body: req.body,
      });
      next();
    } catch (err) {
      next(err);
    }
  };
};
export default validateRequest;
```

Now we can call `validateRequest` any route, before the server execution

```javascript
router.post(
  '/create-student',
  validateRequest(createStudentValidationSchema),
  UserControllers.createStudent,
);
```

or, for another `createAcdemicSemesterValidationSchema` we can reuse the `validateRequest`

```javascript
router.post(
  '/create-academic-semester',
  validateRequest(createAcdemicSemesterValidationSchema),
  AcademicSemesterControllers.createAcademicSemester,
);
```

## Utils vs Middlewares

**Utils:** Utils are the reusable functions used in controllers. The global Util functions will be kept in util folder. And module based util function will be kept in `module_name.utils.ts` file.

**Middlewares:** Middlewares are the function essentially used in http requests. The middleware functions will be kept in `middlewares` folder.

# Part 3 - Branch: `first-project-5`

## Table of Contents

- [Global Error and Not Found Handler (Simplified Example)](#global-error-and-not-found-handler-simplified-example)

## <a id="global-error-and-not-found-handler-simplified-example"></a> Global Error and Not Found Handler (Simplified Example)

```javascript
const express = require('express');
const app = express();

// Route that throws an error
app.get('/', (req, res, next) => {
  const error = new Error('Something went wrong!');
  next(error);
});

// Global 404 Not Found Handler (MUST come after all routes)
app.use((req, res, next) => {
  res.status(404).json({ message: 'Route not found' });
});

// Global Error Handler (MUST have 4 params)r
app.use((err, req, res, next) => {
  res.status(500).json({ message: err.message });
});

// Start the server
app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

#### How Not Found Handler Works:

- If no route matches the request, Express goes to this middleware.
- It catches all unknown routes.
- Sends a `404` status with a `"Route not found"` message.

#### How Error Handler Works:

- The `/` route throws an error using `next(error)`.
- Express skips other middlewares and goes to the error handler.
- The global error handler sends a `500` status with the error message.
