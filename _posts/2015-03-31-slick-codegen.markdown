---
layout: post
title:  "Slick 3 example with code generator"
date:   2015-03-31 00:00:00
tags: [scala, slick]
---

I have recently switched one project to [Slick v3](http://slick.typesafe.com/doc/3.0.0-RC2/). And I wanted to share with you a simple example of an [sbt](http://www.scala-sbt.org/) project using [MySQL](http://www.mysql.com/).
The project contains only one table named `action`.

## MySQL table
{% highlight mysql %}
DROP TABLE IF EXISTS `action`;
CREATE TABLE `action` (
  `id`   BIGINT(20)   NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`id`)
)
  ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
{% endhighlight %}
<i class="fa fa-arrow-right"></i> [view on <i class="fa fa-github-alt"></i>](https://github.com/ArnaudT/slick-codegen/blob/master/src/main/resources/db/Schema.sql)

## SBT generator configuration
The `build.sbt` file looks like the following.
This configuration will generate the slick code from the MySQL tables in a class named `Tables.scala`.

{% highlight scala %}
name := "slickCodeGen"

version := "1.0"

scalaVersion := "2.11.6"

scalacOptions := Seq("-unchecked", "-deprecation", "-encoding", "utf8")

libraryDependencies ++= Seq(
  "com.typesafe.slick" %% "slick" % "3.0.0-RC2",
  "com.typesafe.slick" %% "slick-codegen" % "3.0.0-RC2",
  "mysql" % "mysql-connector-java" % "5.1.34",
  "com.zaxxer" % "HikariCP" % "2.3.2",
  "org.scalatest" %% "scalatest" % "2.2.4" % "test"
)

slick <<= slickCodeGenTask

sourceGenerators in Compile <+= slickCodeGenTask

lazy val slick = TaskKey[Seq[File]]("gen-tables")
lazy val slickCodeGenTask = (sourceManaged, dependencyClasspath in Compile, runner in Compile, streams) map { (dir, cp, r, s) =>
  val outputDir = (dir / "main/slick").getPath
  val username = "user"
  val password = "password"
  val url = "jdbc:mysql://localhost/slick"
val jdbcDriver = "com.mysql.jdbc.Driver"
  val slickDriver = "slick.driver.MySQLDriver"
  val pkg = "com.example.models"
  toError(r.run("slick.codegen.SourceCodeGenerator", cp.files, Array(slickDriver, jdbcDriver, url, outputDir, pkg, username, password), s.log))
  val fname = outputDir + "/" + "me/arnaudtanguy/models" + "/Tables.scala"
  Seq(file(fname))
}
{% endhighlight %}
<i class="fa fa-arrow-right"></i> [view on <i class="fa fa-github-alt"></i>](https://github.com/ArnaudT/slick-codegen/blob/master/build.sbt)

## DAO example with slick 3
You can now use the generated `Tables` class.
I use there [Functional Relational Mapping](http://slick.typesafe.com/doc/3.0.0-RC2/introduction.html#functional-relational-mapping) and [Plain SQL Support](http://slick.typesafe.com/doc/3.0.0-RC2/introduction.html#plain-sql-support) of Slick.

{% highlight scala %}
case class ActionDaoImpl(db: Database) extends ActionDao {
  private implicit val ec = ExecutionContext.fromExecutorService(Executors.newFixedThreadPool(5))

  override def findById(id: Long): Future[Option[Tables.ActionRow]] = {
    val query: Query[Tables.Action, Tables.Action#TableElementType, Seq] = Action.filter(_.id === id)
    val action = query.result.headOption
    db.run(action)
  }

  override def save(name: String): Future[Long] = {
    val query = Action.map(a => (a.name))
    val actions = (for {
      actionInsert <- query += (name)
      actionId <- sql"SELECT LAST_INSERT_ID()".as[(Long)].head
    } yield actionId).transactionally
    db.run(actions)
  }
}
{% endhighlight %}
<i class="fa fa-arrow-right"></i> [view on <i class="fa fa-github-alt"></i>](https://github.com/ArnaudT/slick-codegen/blob/master/src/main/scala/me/arnaudtanguy/dao/ActionDao.scala)

## Tests
The tests use [ScalaTest](http://www.scalatest.org) with the `ScalaFutures` trait to get the future value.

{% highlight scala %}
package me.arnaudtanguy.dao

import com.example.models.Tables.profile.api._
import org.scalatest._
import org.scalatest.concurrent.ScalaFutures
import org.scalatest.time.{Millis, Seconds, Span}

class ActionDaoSpec extends FlatSpec with Matchers with ScalaFutures {
  implicit val defaultPatience = PatienceConfig(timeout = Span(2, Seconds), interval = Span(5, Millis))

  private val db = Database.forConfig("db")
  private val actionDao = ActionDaoImpl(db)

  "save" should "save the value without exception" in {
    actionDao.save("Hello!")
  }

  "findById" should "find a row" in {
    val actionId = actionDao.save("New Hello!").futureValue
    val maybeAction = actionDao.findById(actionId).futureValue

    maybeAction.isDefined should be(true)
    maybeAction.get.name should be("New Hello!")
  }
}
{% endhighlight %}
<i class="fa fa-arrow-right"></i> [view on <i class="fa fa-github-alt"></i>](https://github.com/ArnaudT/slick-codegen/blob/master/src/test/scala/me/arnaudtanguy/dao/ActionDaoSpec.scala)

You can find the project sources on GitHub: [slick-codegen](https://github.com/ArnaudT/slick-codegen).

More information about code generation on the [official Slick documentation](http://slick.typesafe.com/doc/3.0.0-RC2/code-generation.html).

<iframe src="https://ghbtns.com/github-btn.html?user=ArnaudT&repo=slick-codegen&type=fork&count=true" frameborder="0" scrolling="0" width="170px" height="20px"></iframe>
