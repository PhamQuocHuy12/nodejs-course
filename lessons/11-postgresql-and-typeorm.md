# Lesson 11 - PostgreSQL and TypeORM

> Reading time: about 35 minutes  
> Practice time: about 90 minutes

## Outcome

Model relational data, connect Node.js to PostgreSQL with TypeORM, create explicit migrations, use repositories, and protect multi-step changes with transactions.

## Why PostgreSQL and TypeORM

PostgreSQL is the database. It owns durable data, constraints, indexes, and transactions.

TypeORM is an object-relational mapper. It maps JavaScript objects and repository operations to database tables and SQL. It improves developer ergonomics, but it does not remove the need to understand relational modeling or queries.

This course uses TypeORM EntitySchema so the project can remain ESM JavaScript. TypeScript projects often define entities with decorators.

## 1. Relational foundations

A table contains rows. Columns define stored attributes.

~~~sql
CREATE TABLE courses (
  id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title varchar(120) NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
~~~

Important database protections:

- Primary key uniquely identifies a row.
- NOT NULL requires a value.
- UNIQUE prevents duplicates.
- CHECK enforces a condition.
- Foreign key requires a related row.

Application validation improves user messages. Database constraints protect data even if another client or bug bypasses the application.

## 2. Model relationships deliberately

A course has many study sessions. Each session belongs to one course.

~~~sql
CREATE TABLE study_sessions (
  id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  course_id integer NOT NULL REFERENCES courses(id),
  minutes integer NOT NULL CHECK (minutes > 0),
  studied_at timestamptz NOT NULL DEFAULT now()
);
~~~

The foreign key prevents a session from referring to a nonexistent course. Decide what deletion should do:

- RESTRICT refuses deletion while sessions exist.
- CASCADE deletes dependent sessions.
- SET NULL preserves sessions but removes the relationship.

For learning history, RESTRICT is a sensible initial choice.

## 3. Install the database packages

~~~powershell
npm install typeorm pg reflect-metadata
~~~

- typeorm supplies the ORM.
- pg is the PostgreSQL driver.
- reflect-metadata supports TypeORM metadata features.

Set DATABASE_URL through Lesson 08 configuration.

## 4. Define a course EntitySchema

src/entities/course-schema.js:

~~~javascript
import { EntitySchema } from "typeorm";

export const CourseSchema = new EntitySchema({
  name: "Course",
  tableName: "courses",
  columns: {
    id: {
      type: "integer",
      primary: true,
      generated: true,
    },
    title: {
      type: "varchar",
      length: 120,
      nullable: false,
    },
    createdAt: {
      name: "created_at",
      type: "timestamptz",
      createDate: true,
    },
  },
  relations: {
    sessions: {
      type: "one-to-many",
      target: "StudySession",
      inverseSide: "course",
    },
  },
});
~~~

The JavaScript property can be createdAt while the SQL column is created_at. Mapping is useful, but do not let naming hide the database model from you.

## 5. Define the study-session entity

src/entities/study-session-schema.js:

~~~javascript
import { EntitySchema } from "typeorm";

export const StudySessionSchema = new EntitySchema({
  name: "StudySession",
  tableName: "study_sessions",
  columns: {
    id: {
      type: "integer",
      primary: true,
      generated: true,
    },
    minutes: {
      type: "integer",
      nullable: false,
    },
    studiedAt: {
      name: "studied_at",
      type: "timestamptz",
      createDate: true,
    },
  },
  checks: [
    {
      expression: "minutes > 0",
    },
  ],
  relations: {
    course: {
      type: "many-to-one",
      target: "Course",
      inverseSide: "sessions",
      nullable: false,
      onDelete: "RESTRICT",
      joinColumn: {
        name: "course_id",
      },
    },
  },
});
~~~

The relation is modeled on both sides, but the foreign-key column lives with the many-to-one side.

## 6. Configure a DataSource

src/data-source.js:

~~~javascript
import "reflect-metadata";
import { DataSource } from "typeorm";
import { config } from "./config.js";
import { CourseSchema } from "./entities/course-schema.js";
import { StudySessionSchema } from "./entities/study-session-schema.js";

export const AppDataSource = new DataSource({
  type: "postgres",
  url: config.databaseUrl,
  entities: [CourseSchema, StudySessionSchema],
  migrations: ["src/migrations/*.js"],
  synchronize: false,
  logging: false,
});
~~~

Initialize during startup:

~~~javascript
await AppDataSource.initialize();
~~~

synchronize is false because automatic schema synchronization is unsafe as a production migration strategy. Schema changes should be reviewed and versioned.

## 7. Migrations are database history

A migration describes an ordered schema change. Commit migration files with the code that requires them.

Typical CLI shape:

~~~powershell
npx typeorm migration:generate src/migrations/InitialSchema -d src/data-source.js
npx typeorm migration:run -d src/data-source.js
npx typeorm migration:revert -d src/data-source.js
~~~

CLI details can vary with the installed TypeORM version and JavaScript or TypeScript setup. Check the linked official migration guide before configuring package scripts.

Always inspect generated SQL. Generated does not mean reviewed.

## 8. Use a repository behind your application boundary

~~~javascript
export function createTypeOrmSessionRepository(dataSource) {
  const repository = dataSource.getRepository("StudySession");

  return {
    async create({ course, minutes }) {
      const entity = repository.create({ course, minutes });
      return repository.save(entity);
    },

    async findAll() {
      return repository.find({
        relations: {
          course: true,
        },
        order: {
          studiedAt: "DESC",
        },
      });
    },
  };
}
~~~

The service depends on your repository interface, not directly on TypeORM everywhere. The in-memory repository from Lesson 10 can remain useful for fast tests.

## 9. Transactions protect a unit of work

Suppose completing a session must insert the session and update a course total. Both changes should succeed or both fail.

~~~javascript
await dataSource.transaction(async (manager) => {
  const sessionRepository = manager.getRepository("StudySession");
  const courseRepository = manager.getRepository("Course");

  await sessionRepository.save(
    sessionRepository.create({ course, minutes }),
  );

  await courseRepository.increment(
    { id: course.id },
    "totalMinutes",
    minutes,
  );
});
~~~

Use the transaction-provided manager for every query inside the transaction. Using a global repository there can accidentally escape the transaction.

## 10. Indexes are a tradeoff

If sessions are frequently listed by course and date:

~~~sql
CREATE INDEX study_sessions_course_date_idx
ON study_sessions (course_id, studied_at DESC);
~~~

An index may speed reads that match it, but it consumes storage and adds work to writes. Add indexes from query needs and measured evidence, not as decorative database confetti.

## Real-world applications and edge cases

### Where this appears

- Order and payment records requiring transactions
- Multi-tenant ownership enforced by query conditions and constraints
- Reporting queries over growing history tables
- Rolling deployments that change schemas without stopping traffic

### Edge cases to investigate

- Two requests check that an email is free, then both insert. Only a unique constraint resolves the race correctly.
- Transactions update rows in different orders and deadlock; one transaction must be retried.
- A long transaction holds locks while waiting for an external API.
- A loop loads one relationship per row and creates an N+1 query problem.
- Every application instance opens a large connection pool and collectively exhausts PostgreSQL.
- Offset pagination becomes slow and unstable while new rows are inserted.
- A timestamp loses timezone meaning or is interpreted in the server's local zone.
- Automatic retries repeat a transaction with a non-database side effect such as sending email.
- A migration renames or removes a column while old and new application versions overlap.
- Soft-deleted rows still participate in uniqueness rules unexpectedly.

Database correctness lives in constraints, transaction boundaries, isolation behavior, query plans, and deployment compatibility—not only ORM entity definitions.

## Pause and predict

- Which layer protects against a session with a nonexistent course?
- What could happen if synchronize is enabled against production data?
- Why must transaction queries use the supplied manager?
- Does an ORM make SQL performance irrelevant?

## Guided practice

1. Run PostgreSQL locally or provision a development database.
2. Configure DATABASE_URL.
3. Define Course and StudySession schemas.
4. Generate and review a migration.
5. Run the migration.
6. Replace in-memory repositories with TypeORM repositories.
7. Restart the server and verify that data remains.
8. Inspect the actual tables and constraints in PostgreSQL.

## Common mistakes

- Enabling synchronize in production
- Changing an entity without creating a migration
- Trusting application validation without database constraints
- Returning database entities with sensitive or internal fields
- Fetching relationships repeatedly and creating an N+1 query problem
- Creating indexes without understanding the query
- Using global repositories inside a transaction callback
- Catching every database error as a generic 500

## Independent challenge

Add users to the model:

- A user owns many courses.
- Course titles must be unique per user, not globally.
- Sessions remain connected through their course.
- Deleting a user must use a deliberate retention rule.

Then implement a report returning total study minutes per course for a date range. Write the SQL query you expect before expressing it with TypeORM.

## Knowledge check

- What protection does a foreign key provide?
- Why are migrations committed?
- What responsibility remains with PostgreSQL?
- When is a transaction necessary?
- Why keep a repository boundary around TypeORM?

## Explore with an AI agent

- Review my entity schemas against the SQL constraints I intended.
- Explain the SQL generated by my TypeORM query and identify possible N+1 behavior.
- Give me a failed migration scenario and ask me to plan recovery.
- Compare EntitySchema in JavaScript with decorators in TypeScript using the same Course model.

## Official reading

- [PostgreSQL tutorial](https://www.postgresql.org/docs/current/tutorial.html)
- [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [TypeORM getting started](https://typeorm.io/docs/getting-started/)
- [TypeORM EntitySchema](https://typeorm.io/docs/entity/separating-entity-definition/)
- [TypeORM migrations](https://typeorm.io/docs/advanced-topics/migrations/)
- [TypeORM transactions](https://typeorm.io/docs/advanced-topics/transactions/)
- [TypeORM PostgreSQL driver](https://typeorm.io/docs/drivers/postgres/)

## Definition of done

- PostgreSQL preserves data across application restarts.
- Migrations can create a fresh schema.
- Constraints enforce core data integrity.
- TypeORM stays behind repository boundaries.
- You can explain the SQL model beneath the entities.
