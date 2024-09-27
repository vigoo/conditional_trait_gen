# conditional_trait_gen

A fork of the [trait_gen](https://github.com/blueglyph/trait_gen) crate, adding support for specializing trait methods
based on the type it is generated for.

Introduces a `#[when]` attribute that can be used on methods of the trait implementation to customize the method based on
the actual type used by the generator.

Example:

```rust
 #[async_trait]
trait Repo {
    async fn create(&self, param: String) -> Result<u64, String>;
    async fn read(&self, id: u64) -> Result<String, String>;
    async fn update(&self, id: u64, param: String) -> Result<(), String>;
    async fn delete(&self, id: u64) -> Result<(), String>;
}

#[trait_gen(DB -> sqlx::sqlite::Sqlite, sqlx::mysql::MySql, sqlx::postgres::Postgres)]
#[async_trait]
impl Repo for sqlx::Pool<DB> {
    async fn create(&self, param: String) -> Result<u64, String> {
        // Common implementation
        todo!()
    }

    async fn read(&self, id: u64) -> Result<String, String> {
        // Common implementation
        todo!()
    }

    #[when(sqlx::sqlite::Sqlite -> update)]
    async fn update_sqlite(&self, id: u64, param: String) -> Result<(), String> {
        // SQLite specific implementation
        todo!()
    }

    #[when(sqlx::mysql::MySql -> update)]
    async fn update_mysql(&self, id: u64, param: String) -> Result<(), String> {
        // MySQL specific implementation
        todo!()
    }

    #[when(sqlx::postgres::Postgres -> update)]
    async fn update_postgres(&self, id: u64, param: String) -> Result<(), String> {
        // Postgres specific implementation
        todo!()
    }

    async fn delete(&self, id: u64) -> Result<(), String> {
        // Common implementation
        todo!()
    }
}
```