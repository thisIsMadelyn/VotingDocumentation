## Voting App Technical Requirements
### 1. Introduction
 When developing a web service of this scale the challenge isn't just to avoid syntax and runtime errors, it's organizing the directories in a way that is easy to maintain by the developers after you. 
 This guide provides a structured approach to creating and mainting the API for the LC.

### 2. Document Structure Overview
- Executive Summary
- System Architecture
- API Organization & Structure
- Functional Requirements 
- Non-functional Requirement
- Data Model & Schema
- Authentication & Authorization
- Error Handling & Status Codes
- Testing Requirements
- Integration Requirements
- Deployment & Operations
- Appendices 

### 3. Executive Summary
**Purpose:**
The problem this API solves for the LC is the need for a robust and secure voting Web App. The primary consumers are the members of the Board and thee Supervisory Board as well as the LC's members that are allowed to vote during General Meetings.  

**Scope & Scale:**
The number of endpoints is estimated around: 100. *API VESRIONING STRATEGY NOT DECIDED YET.*

### 4. System Archritecture
**High Level Architecture:**
API type REST. Monolithic Structure (One Database, one API). The monolithic structures in our case benefits the team immensely because it reduces the need for complex DevOps (Developement Operation Teams). The layers of the API are (Entity, Repository -> Service -> Controller, Validator, Authentication).

**Component Diagram:**
*TO ADD ER(ENTITY RELATIONSHIP) DIAGRAM, SHORT*

**Technology Stack:**
- Java EE (Enterprise Edition), Spring Boot, JPA (Hibernate), Spring Security (JWT)
- SQL, MySQL DBMS (DataBase Management System) _for local environment_
- HTTP requests, HTTP sessions
- *CLOUD INFRASTRUCTURE TO BE DECIDED*

### 5. API Organization & Structure
```
main /
| controllers/
| services/
| repositories/
| entities/
| config/
| exceptions/
| enums/
| DTOs/
```
**Naming Conventions & Rules:**
- Camel Case: ``` thisIsAClass.java ```
- Controllers: ```usercontroller.java``` etc
- Services: ```userService.java``` etc
- REpositories: ```userRepository``` etc
- DTOs(Data Transfer Objects): ```helloDTO.java``` etc

**Module Grouping:**
We will be grouping related files in directories or packages (in java) as mentioned above, in the API Organization structure and NOT as they are organized in tables in the DB (User: Entity, Repository, Service, Controller, Poll: Entity,..).

**URL Structure & Versioning:**
This paragraph contains the configuration of the endpoints. _TO ADD ENDPOINTS_.

### 6. Functional Requirements

This is  where the core section of this document begins.
**Endpoint Documentation Template:** (example), _TO ADD ALL ENDPOINTS_
- Endpoint ID: Unique identifier
- HTTP Method:GET
- URL Path: ```/api/users/user/username/{username}```
- Purpose: It gets a user that is searched in the DB by username
- Status Code: 200 OK.

**Scalability Requirements** _TO BE ADDED._

**Security Requirements**
- Authentication: JWT tokens etc.
- Authorization: Role-based acces Control (RBAC)
- Data Encryption: _TO BE ADDED_

### 8. Data Models & Schemas

**Entity Relationship Diagram:** _IN DETAIL_

**JSON Schema Definitions:**

### 9. Authentication & Authorization
