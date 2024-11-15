# NoSQL Brainiac Part 1 - Branch: `first-project-3`
### Requirement Analysis

[Link to Requirement Analysis Document](https://docs.google.com/document/d/10mkjS8boCQzW4xpsESyzwCCLJcM3hvLghyD_TeXPBx0/edit?usp=sharing)

Description: This document outlines the detailed analysis of project requirements.

---

### Entity-Relationship Diagrams

![ER DIAGRAM](./erdiagram.png)

Description: This is an updated diagram illustrates the relationships among User, Student, Admin, Faculty, Academic Semester, Academic Faculty, Academic Department , Course , Semester Registration , Offered Couse.

---

![POSTMAN COLLECTION](./postman_collection.json)

Description: This is a postman collection of all the API endpoints.Download this , and import it in your postman if you needed.

---
# Part 1 - Branch: `first-project-3`
## Table of Contents
- [Introduction](#introduction)
- [Mongoose: Static vs Method](#mongoose-static-vs-method)

## Introduction
**SQL:** Sequential Query Language (Oracle, MySQL). Collection, Document, Field

**NoSQL:** No Sequential Query Language (mogoDB, mariaDB, Radis, DynamoDB) Table, Row, Column(Field)

## Mongoose: Static vs Method

In Mongoose, **statics** and **methods** serve different purposes despite both being used to define reusable functions for schemas. The main difference lies in **how** they are used and the **context** in which they operate.

| **Feature** | **Statics** | **Methods** |
| --- | --- | --- |
| **Context** | Operates on the **model/class level** (e.g., `Student`). | Operates on the **instance/document level** (e.g., a specific student). |
| **Use Case** | For operations that do not require a specific document (e.g., queries, aggregations, or utilities). | For operations related to a specific document (e.g., modifying a property, checking state). |
| **Access** | Accessed via the **model** (e.g., `Student.findByAge(age)`). | Accessed via the **instance** (e.g., `studentInstance.isAdult()`). |

#### **When to Use** `Statics`

Use `statics` when the operation involves the **entire collection** or the model as a whole, and does not pertain to a specific document.

#### Example: Static Method for Finding Documents by Age

```plaintext
studentSchema.statics.findByAge = async function (age) {
  return await this.find({ age });
};

// Usage:
const students = await Student.findByAge(18);
```

**When to Use** `Methods`

Use `methods` when the operation involves **an individual document** or needs to modify/work with specific document fields.

#### Example: Method to Check if a Student is an Adult

```plaintext
studentSchema.methods.isAdult = function () {
  return this.age >= 18;
};

// Usage:
const student = await Student.findOne({ name: 'John' });
console.log(student.isAdult()); // true or false
```

#### **Key Decision Criteria**

1.  **Does the function involve one document or many?**
    *   **One Document:** Use a `method`.
    *   **Multiple Documents or the Model Itself:** Use `statics`.
2.  **Do you need access to instance properties (**`**this**`**) like** `**this.age**` **or** `**this.name**`**?**
    *   **Yes:** Use a `method`.
    *   **No:** Use `statics`.
3.  **Is the operation generic to the model or specific to an instance?**
    *   **Generic:** Use `statics`.
    *   **Specific to an Instance:** Use `methods`.


    