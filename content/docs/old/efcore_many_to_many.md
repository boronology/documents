+++
draft = false
title = 'Entity Framework Coreでの多対多の扱い方について'
+++

# Entity Framework Coreでの多対多の扱い方について

Entity Framework Core（以下、EFCore）は.NETにおけるMicrosoft製のO/R
mapperである。EFCoreは.NETのクラスとして宣言されたエンティティをデータベースのテーブルにマッピングし、別途定義されたナビゲーションにしたがってリレーションを決定する。さらにEFCoreはエンティティとナビゲーションにしたがって.NETのLINQやプロパティアクセスをSQLに変換する。

この記事で扱うのはEFCoreにおける多対多の関連の扱い方である。もっともMicrosoft
Learn上には多対多リレーションシップに関するドキュメントがある（[多対多リレーションシップ EF Core Microsoft Learn](https://learn.microsoft.com/ja-jp/ef/core/modeling/relationships/many-to-many)）。しかしこのドキュメントはテーブルとリレーションの定義方法について扱っているのみで、実際にCRUDの操作を行う方法についての記述はほとんどない。本記事では具体的な操作方法と要点をまとめる。

## サンプルプロジェクト

以下のコードはサンプルプロジェクトからの抜粋である。プロジェクトは以下のURLにアップしている（
[boronology/EfcoreTest: EntityFrameworkCoreの多対多サンプル](https://github.com/boronology/EfcoreTest)
）。

## テーブルとリレーション

本記事では上記Microsoft Learnの記事と同様に、 `Post` （ブログの記事）と `Tag`
（その記事に付けられたタグ）の多対多の関連を例とする。

エンティティは以下のように定義した。

```cs
[Table("posts")]
[PrimaryKey(nameof(PostId))]
class DbPost
{
    [Column("post_id")]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public Guid PostId { get; set; }
    [Column("title")]
    public string Title { get; set; }

    public List<DbTag> Tags { get; set; } = [];
    public List<DbPostTag> PostTags { get; set; } = [];
}

[Table("tags")]
[PrimaryKey(nameof(TagId))]
class DbTag
{
    [Column("tag_id")]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public Guid TagId { get; set; }
    [Column("name")]
    public string Name { get; set; }

    public List<DbPostTag> PostTags { get; } = [];
    public List<DbPost> Posts { get; } = [];
}

[Table("post_to_tag")]
class DbPostTag
{
    [Column("post_id")]
    [ForeignKey(nameof(DbPost))]
    public Guid PostId { get; set; }
    [Column("tag_id")]
    [ForeignKey(nameof(DbTag))]
    public Guid TagId { get; set; }

    public DbPost Post { get; }
    public DbTag Tag { get; }
}
```

さらにContextを以下のように定義する。

```cs
abstract class DataBaseContext : DbContext
{
    public DbSet<DbPost> Posts { get; set; }
    public DbSet<DbTag> Tags { get; set; }
    public DbSet<DbPostTag> PostTags { get; set; }

    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        base.ConfigureConventions(configurationBuilder);
        configurationBuilder.Properties<Guid>().HaveConversion<GuidToStringConverter>();
    }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        //多対多のリレーションを明示的に指定
        modelBuilder.Entity<DbPost>()
            .HasMany(e => e.Tags)
            .WithMany(e => e.Posts)
            .UsingEntity<DbPostTag>
            (
                l => l.HasOne(e => e.Tag).WithMany(e => e.PostTags).HasForeignKey(e => e.TagId),
                r => r.HasOne(e => e.Post).WithMany(e => e.PostTags).HasForeignKey(e => e.PostId),
                j => j.HasKey(e => new { e.PostId, e.TagId })
            );
    }
}
```

ここでは `OnModelCreating()`
で多対多のリレーションを明示的に指定している。Microsoftのドキュメントでは明示的な構成を避けるようにと言っているが、本気に受け止めないほうがいい。少し複雑な操作に足を踏み入れるとどのみちSQLを読みながらモデルを操作することになるからだ。

> 必要ない場合でも、すべてを完全に構成しようとしないでください。
> 上記のように、コードはすぐに複雑になり、間違いが発生しやすくなります。
> さらに、上記の例でも、モデルには規則によって構成されるものが多くあります。 EF
> モデル内のすべてを常に明示的に完全に構成できると考えるのは、現実的ではありません。

EFCoreは上記のコードをおおよそ以下のSQLに変換する。

```sql
-- SQLiteの場合
CREATE TABLE IF NOT EXISTS "posts" (
    "post_id" TEXT NOT NULL CONSTRAINT "PK_posts" PRIMARY KEY,
    "title" TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS "tags" (
    "tag_id" TEXT NOT NULL CONSTRAINT "PK_tags" PRIMARY KEY,
    "name" TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS "post_to_tag" (
    "post_id" TEXT NOT NULL,
    "tag_id" TEXT NOT NULL,
    CONSTRAINT "PK_post_to_tag" PRIMARY KEY ("post_id", "tag_id"),
    CONSTRAINT "FK_post_to_tag_posts_post_id" FOREIGN KEY ("post_id") REFERENCES "posts" ("post_id") ON DELETE CASCADE,
    CONSTRAINT "FK_post_to_tag_tags_tag_id" FOREIGN KEY ("tag_id") REFERENCES "tags" ("tag_id") ON DELETE CASCADE
);
```

## クエリ

`Post` をもとに、それに紐づく `Tags` を操作するユースケースを考える。

## Read

取得についてはとくに説明は不要だろう。ナビゲーションを正しく定義していれば以下のように
`Include()` で目的のテーブルを `JOIN` して取得できる。

```cs
using var context = GetDataBaseContext();
var allPosts = context.Posts.Include(e => e.Tags);
```

発行されるクエリはおおよそ以下のようになるはずだ。

```sql
SELECT p.title, p.post_id, t0."Name", t0."TagId", t0.post_id, t0.tag_id
FROM posts AS p
LEFT JOIN (
    SELECT t.name AS "Name", t.tag_id AS "TagId", p0.post_id, p0.tag_id
    FROM post_to_tag AS p0
    INNER JOIN tags AS t ON p0.tag_id = t.tag_id
) AS t0 ON p.post_id = t0.post_id
ORDER BY p.post_id, t0.post_id, t0.tag_id
```

### Delete

`Post` から `Tag` を削除する場合。中間テーブルに対して `ExecuteDelete()`
を使うのが基本となる。これはすべての `Tag` を削除する場合も、一部の `Tag`
のみを指定して削除する場合も同じである。

すべてのタグを削除するとき。発行されるクエリはコードから予想される通り。

```cs
static void DeleteAllTags(Guid postId)
{
    using var context = GetDataBaseContext();
    context.PostTags
        .Where(e => e.PostId == postId)
        .ExecuteDelete();
}
```

```sql
DELETE FROM "post_to_tag" AS "p" WHERE "p"."post_id" = @__postId_0
```

こちらは一部のみ削除するとき。`Contains()` が `IN`
になってほしいところだがすこし不思議なクエリになる。

```cs
static void DeleteSomeTags(Guid postId, IEnumerable<Guid> deleteTags)
{
    using var context = GetDataBaseContext();
    context.PostTags
        .Where(e => e.PostId == postId && deleteTags.Contains(e.TagId))
        .ExecuteDelete();
}
```

```sql
-- SQLite
DELETE FROM "post_to_tag" AS "p"
WHERE "p"."post_id" = @__postId_0 AND "p"."tag_id" IN (
    SELECT "d"."value"
    FROM json_each(@__deleteTags_1) AS "d"
)
```

```sql
-- PostgreSQL
DELETE FROM post_to_tag AS p
WHERE p.post_id = @__postId_0 AND p.tag_id = ANY (@__deleteTags_1)
```

### Create

`Post` に紐づく `Tag` を新たに作成する場合。

もっとも単純な（そして汎用性が高い）操作は「読んで、変更して、保存する」というEFCoreの基本的な手順である。以下の例では対象の
`Post` を取得し、 `Post.Tags.AddRange()` で要素を追加し、 `SaveChanges()`
で保存している。

```cs
static void AddNewPostTags(Guid postId, IEnumerable<Guid> tagIds)
{
    using var context = GetDataBaseContext();
    var newTags = tagIds.Select(id => new DbTag { TagId = id, }).ToArray();
    context.AttachRange(newTags);
    var post = context.Posts.Include(e => e.Tags).FirstOrDefault(e => e.PostId == postId);

    post.Tags.AddRange(newTags);
    context.SaveChanges();
}
```

実際には一度取得する必要は（あまり）ない。追加しようとしている `Tag` と `Post`
の関連がまだ存在しないのであれば直接 `Add()`
できる。一意性制約に引っかからないからだ。

まず、 `Post` を取得するにあたって `Include()` で `Tags` や `PostTags`
を取得する必要はない。これで十分だ。

```cs
static void AddToTags(Guid postId, IEnumerable<Guid> addTagIds)
{
    using var context = GetDataBaseContext();
    var tags = addTagIds.Select(id => new DbTag { TagId = id }).ToArray();
    context.AttachRange(tags);

    var post = context.Posts.FirstOrDefault(e => e.PostId == postId);
    post.Tags.AddRange(tags);
    context.SaveChanges();
}
```

`Post` を取得する必要すらない。中間テーブル `context.PostTags`
に直接追加すればよい。

```cs
static void AddToPostTags(Guid postId, IEnumerable<Guid> addTagIds)
{
    using var context = GetDataBaseContext();

    var postTags = addTagIds.Select(id => new DbPostTag { PostId = postId, TagId = id });
    context.PostTags.AddRange(postTags);

    context.SaveChanges();
}
```

いずれの場合も追加するレコードの数だけ `INSERT` が発行される。
[その他の変更の追跡の機能 EF Core Microsoft Learn](https://learn.microsoft.com/ja-jp/ef/core/change-tracking/miscellaneous#addrange-updaterange-attachrange-and-removerange)にあるとおり、
`AddRange()` は `Add()` の複数回呼び出しと同じだからだ。

### Update

Updateと括っているが、この節で扱うのは「登録しようとしている `Tag` と `Post`
の関連がすでに存在する（するかもしれない）場合」だ。ユースケースとしては以下を考える。

- `Post` に紐づく `Tag`
  をすべて付け替える。もちろん付け替え前と付け替え後に重複がありうる。
- ある `Tag` がすでに `Post` に紐づいているかどうかにかかわらず紐づけさせる

#### すべて付け替える場合

`Tag` をすべて付け替える場合は一度 `Post` の取得が必要だ。取得した `Post` の
`Post.PostTags` を変更して `SaveChanges()` する。なお、ここでの `post.PostTags`
への操作は `post.Tags` の操作にも置き換えられる。

```cs
static void ReplaceTags(Guid postId, IEnumerable<Guid> newTags)
{
    using var context = GetDataBaseContext();

    //一度取得
    var post = context.Posts.Include(e => e.PostTags).FirstOrDefault(e => e.PostId == postId);

    var postTags = newTags.Select(id => new DbPostTag { TagId = id }).ToList();

    //1. すべて削除してあらためて追加
    post.PostTags.Clear();
    post.PostTags.AddRange(postTags);

    //2. リストごと入れ替え
    //post.PostTags = postTags;

    context.SaveChanges();
}
```

この処理では発行されるクエリは以下のようになる。入れ替え後に消える `Tag`
2つに対してそれぞれ `DELETE` が、追加される `Tag` 2つに対して `INSERT`
が発行されている。入れ替え前後で変化のない1つは触れられていない。

```sql
-- 取得
SELECT t.post_id, t.title, p0.post_id, p0.tag_id
FROM (
    SELECT p.post_id, p.title
    FROM posts AS p
    WHERE p.post_id = @__postId_0
    LIMIT 1
) AS t
LEFT JOIN post_to_tag AS p0 ON t.post_id = p0.post_id
ORDER BY t.post_id, p0.post_id

-- 保存
DELETE FROM post_to_tag
WHERE post_id = @p0 AND tag_id = @p1;
DELETE FROM post_to_tag
WHERE post_id = @p2 AND tag_id = @p3;
INSERT INTO post_to_tag (post_id, tag_id)
VALUES (@p4, @p5);
INSERT INTO post_to_tag (post_id, tag_id)
VALUES (@p6, @p7);
```

#### 重複を考慮して追加する場合

重複の可能性がある `Add()`
の場合、残念ながら自分で重複を弾く必要がある。愚直に1要素ずつ見てもよいが、
`HashSet<T>`
で一意にするのが楽だろう。当然のことながら新たに追加される要素のぶんだけ
`INSERT` が発行される。

```cs
static void AddWithDuplicate(Guid postId, IEnumerable<Guid> addTagIds)
{
    using var context = GetDataBaseContext();
    var post = context.Posts.Include(e => e.PostTags).FirstOrDefault(e => e.PostId == postId);

    var newTagIds = new HashSet<Guid>(addTagIds);
    newTagIds.UnionWith(post.PostTags.Select(e => e.TagId));
    var newPostTags = newTagIds.Select(id => new DbPostTag { TagId = id });
    post.PostTags = newPostTags.ToList();

    context.SaveChanges();
}
```

## まとめ

多対多のリレーションであってもEFCoreの基本的なコンセプトは「追跡」である。EFCoreは単純にデータベースと.NETオブジェクトを相互に変換しているだけではない。
`SaveChanges()`
でEFCoreは把握しているデータベースの状態とオブジェクトの状態を比較し、その差を埋めるようなクエリを発行する。レコードの作成や変更のときには常にこの動作を意識しなければならない。

基本は「読んで、変更して、保存する」である。データベースから読み込むとEFCoreがデータの追跡を始める。取得したオブジェクトを変更する。保存するとEFCoreが読み込んだデータとの差分を調べ、クエリを発行する。データベースから読まずに変更や追加ができるのは比較元がなくても結果が変わらない特殊なパターンと考えるほうがよい。

EFCoreをクエリビルダーとしてとらえてはいけない。思い通りのクエリを発行したかったら
`FromSqlRaw()` を使うべきだし、もっと言うならEFCoreを使うべきではない。
