# quill-trait

Support for using of annotated case classes with arbitrary number of traits as 
[Quill](http://getquill.io/) domain models

[![Build Status](https://travis-ci.org/nstojiljkovic/quill-trait.svg?branch=master)](https://travis-ci.org/nstojiljkovic/quill-trait)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/c48cf4b7fa9a4772acbc7134d51f2681)](https://www.codacy.com/app/nstojiljkovic/quill-trait?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=nstojiljkovic/quill-trait&amp;utm_campaign=Badge_Grade)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.github.nstojiljkovic/quill-trait_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.github.nstojiljkovic/quill-trait_2.11)

This documentation assumes you are already familiar with [Quill](http://getquill.io/) 
and that you have read its [documentation](http://getquill.io/).

## What does it do?

The main goal of this library is to allow extending of base domain models using traits. The requirement come from
developing a rapid-application-development-framework/content-management-system where core package's domain models 
should be easily extendable by a a 3rd party libraries.

### Use annotations to define table/column names

Sample:

```scala
import com.nikolastojiljkovic.annotation.{ Field, Table }

@Table("page")
case class Page(
  @Field("id") id: Option[Int],

  @Field("parent_id") parentId: Option[Int],

  @Field("sorting") sorting: Option[Int],

  @Field("title") title: String,

  @Field("path") path: String
)
```

### Use annotated traits

Sample: 

```scala
@Table("page")
trait SeoSupport {
  this: Page =>

  @Field("sitemap_change_freq")
  def sitemapChangeFrequency: String

  @Field("sitemap_priority")
  def sitemapPriority: Double
}

```

In the sample above, usage of `Table` annotation is not required.

A trait without a `Table` annotation can be used with any case class (as long as the mapped table 
has all of the required fields).

Few restrictions for the traits:
* The only abstract methods need to be the ones mapped to a database column,
* Counter-intuitive `def` needs to be used in traits to define columns (instead of `val`) as 
  Scala compiler doesn't expose annotations of trait's `val` members. Internally, when instantiating,
  those `def`s will actually become `val`.
  
### Usage of annotated case classes and traits

Any Quill context (MySQL, PostgreSQL, Cassandra...) can be used as long as trait 
`AnnotatedTraitSupport` is included.

```scala
import io.getquill._
import com.nikolastojiljkovic.quilltrait._

val ctx = new SqlMirrorContext[MirrorSqlDialect, Literal] with AnnotatedTraitSupport

import ctx._
```

As table and column names are defined in annotations, it is recommended to use appropriate `*Escape`
naming strategy designated for the selected Quill context.

Context with trait `AnnotatedTraitSupport` can accept annotated case classes and traits the same way
core Quill works with case classes without any limitations.

#### Some samples

Select with filtering:

```scala
val q = quote {
  query[Page with SeoSupport].filter(p => p.sitemapPriority > 0.7)
}

ctx.run(q)
```

Inserting:

```scala
val page = new Page(Some(1), None, Some(0), "Root", "/") with SeoSupport {
  val sitemapChangeFrequency = "monthly"
  val sitemapPriority = 0.00
}
val pages = quote {
  query[Page with SeoSupport]
}
val pageInsert = quote {
  (p: Page with SeoSupport) => pages.insert(p)
}
ctx.run(pageInsert(lift(page)))
```

Updating:

```scala
val u = quote {
  query[Page with SeoSupport].filter(p => p.id == lift(Option(1))).update(_.sitemapPriority -> lift(0.1))
}

ctx.run(u)
```

Deleting:

```scala
val d = quote {
  query[Page with SeoSupport].filter(p => p.id == lift(Option(1))).delete
}

ctx.run(d)
```

### Installation

Add `quill-trait-core` to sbt dependencies:

```
libraryDependencies ++= Seq(
  "com.github.nstojiljkovic" %% "quill-trait-core" % "0.1.3-SNAPSHOT"
)
```

Library `quill-trait-core` works with any Quill context (MySQL, PostgreSQL, JDBC, Cassandra, even Scala.js).

## How does it work?

Quill allows customizing of expansion and execution handling of quotations through implicit 
meta instances (schema, insert, update and query meta instances). Internally, core Quill's 
contexts have defined low priority implicits for those meta instances which are executed 
using macro expansion. Quill-trait context trait `AnnotatedTraitSupport` defines higher priority 
implicits (higher priority than core's low priority implicits) which define meta instances based 
on the annotations.

## Future improvements

* Improve documentation
* Write more test cases
* Move annotations to a separate library
* Create macro for using copy with Quill-core annotated traits

## Contributing

See the [CONTRIBUTING.md](https://github.com/nstojiljkovic/quill-trait/blob/master/CONTRIBUTING.md) file for details.

## License

See the [LICENSE](https://github.com/nstojiljkovic/quill-trait/blob/master/LICENSE) file for details.

## Maintainers

- @nstojiljkovic