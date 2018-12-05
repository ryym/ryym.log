+++
date = "2018-12-02T17:54:10+09:00"
description = ""
draft = false
tags = ["rust"]
title = "Diesel で Enum を使いたい時 (2018年版)"
updated = "2018-12-02T17:54:10+09:00"
+++

## やりたい事

- [Diesel](diesel.rs)でカラムの型に Enum を使いたい
- diesel 1.3.3 で有効

以下のブログで、カラムの型に Enum を使いつつ DB の Integer にマッピングするマクロが紹介されています。

<https://keens.github.io/blog/2017/12/16/dieselshounetashuu/>

しかし diesel 1.3.3 ではこの記事の頃とはインターフェイスが変わっているようで、少し修正が必要でした。
なので 2018 年版の方法としてメモしておきます。

## 方法

記事のマクロと基本は一緒ですが、以下の変化があります:

- `FromSqlRow`が derive できるようになった。
- `AsExpression`が derive できるようになった。
    - `expression_impls!`マクロは削除された ([diesel-rs/diesel#1453](https://github.com/diesel-rs/diesel/pull/1453))。
- `diesel::types`モジュールが [1.1.0][diesel-changelog-1.1.0] から deprecated になった。

[diesel-changelog-1.1.0]: https://github.com/diesel-rs/diesel/blob/master/CHANGELOG.md#110---2018-01-15

`FromSqlRow`の実装が不要になった分、少し短くなりました。

```rust
macro_rules! define_enum {
    (
        $(#[$meta:meta])*
        pub enum $name:ident { $($variant:ident = $val:expr,)* }
    ) => {
        use diesel::sql_types::Integer;
        use diesel::serialize::ToSql;
        use diesel::deserialize::FromSql;

        // 元の enum を必要な derive とともに定義
        $(#[$meta])*
        #[derive(FromSqlRow, AsExpression)]
        #[sql_type = "Integer"]
        pub enum $name {
            $($variant = $val,)*
        }

        // `ToSql`を定義
        impl<DB: diesel::backend::Backend> ToSql<Integer, DB> for $name {
            fn to_sql<W: std::io::Write>(
                &self,
                out: &mut diesel::serialize::Output<W, DB>,
            ) -> Result<diesel::serialize::IsNull, Box<std::error::Error + Send + Sync>> {
                ToSql::<Integer, DB>::to_sql(&(*self as i32), out)
            }
        }

        // `FromSql`を定義
        impl<DB: diesel::backend::Backend> FromSql<Integer, DB> for $name
        where
            i32: FromSql<Integer, DB>,
        {
            fn from_sql(
                bytes: Option<&DB::RawValue>,
            ) -> Result<Self, Box<std::error::Error + Send + Sync>> {
                use self::$name::*;

                match <i32 as FromSql<Integer, DB>>::from_sql(bytes)? {
                    $($val => Ok($variant),)*
                    s => Err(format!("invalid {} value: {}", stringify!($name), s).into()),
                }
            }
        }
    }
}
```

元記事では`Queryable`も impl してましたが、これも不要になったようです。

使用例 (元記事と同じ):

```rust
define_enum! {
    #[derive(Debug, Clone, Copy)]
    pub enum Visibility {
        Public = 0,
        Limited = 1,
        Private = 2,
    }
}

#[derive(Queryable)]
pub struct Post {
    pub id: PostId,
    pub title: String,
    pub body: String,
    pub published: Visibility,
}
```
