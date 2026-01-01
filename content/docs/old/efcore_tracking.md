+++
draft = false
title = 'Entity Framework Coreでの追跡と関連データの読み込みについて'
+++

# Entity Framework Coreでの追跡と関連データの読み込みについて

前回（[Entity Framework Coreでの多対多の扱い方について](https://boronology.github.io/documents/efcore_many_to_many)）に続き、今回もEntityFrameworkCore（以下、EFCore）の記事である。

EFCoreのコンセプトは「追跡」である。追跡はEFCoreにより自動で行われるため、使っていると時折不思議な挙動を目にすることがある。この記事では追跡により
`Include()` なしでも関連データが読み込まれることがある点を扱う。

## サンプルコード

サンプルコードは以下のURLにアップロードしている（[GitHub - boronology/EfcoreProp](https://github.com/boronology/EfcoreProp)）。

### テーブル定義

例として `DbPublisher` （出版社）と `DbBook`
（本）の1対多の関連を扱う。以下のようにエンティティを定義する。

```cs
[Table("publishers")]
[PrimaryKey(nameof(PublisherId))]
class DbPublisher
{
    [Column("publisher_id")]
    public Guid PublisherId { get; set; }

    [Column("name")]
    public string Name { get; set; }

    public List<DbBook> Books { get; set; } = [];
}

[Table("books")]
[PrimaryKey(nameof(BookId))]
class DbBook
{
    [Column("book_id")]
    public Guid BookId { get; set; }

    [Column("title")]
    public string Title { get; set; }

    [Column("publisher_id")]
    [ForeignKey(nameof(DbBook))]
    public Guid PublisherId { get; set; }


    public DbPublisher Publisher { get; set; }
}
```

### リレーションの宣言

前回同様、リレーションは明示的に指定する。

```cs
class DatabaseContext : DbContext
{
    public DbSet<DbBook> Books { get; set; }

    public DbSet<DbPublisher> Publishers { get; set; }
    protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
    {
        base.ConfigureConventions(configurationBuilder);
        configurationBuilder.Properties<Guid>().HaveConversion<GuidToStringConverter>();
    }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<DbBook>()
            .HasOne(e => e.Publisher)
            .WithMany(e => e.Books)
            .HasForeignKey(e => e.PublisherId);
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        base.OnConfiguring(optionsBuilder);
        var builder = new SqliteConnectionStringBuilder()
        {
            ForeignKeys = true,
            DataSource = "database.db",
        };
        optionsBuilder.UseSqlite(builder.ToString());
    }
}
```

## 問い合わせ

DBから本の情報を取得することを考える。取得したデータは以下のようにダンプする。

```cs
private static void ShowBook(DbBook dbBook)
{
    Console.WriteLine($"BookId        :{dbBook.BookId}");
    Console.WriteLine($"Title         :{dbBook.Title}");
    Console.WriteLine($"PublisherId   :{dbBook.PublisherId}");
    Console.WriteLine($"PublisherName :{dbBook?.Publisher?.Name ?? "(NULL)"}");
}
```

### DbBookを1件取得

もっともシンプルな `DbBook` を取得するコードは以下である。

```cs
private static async Task GetOneBook()
{
    Console.WriteLine("* DBからDbBookを1件取得");
    using var context = new DatabaseContext();
    var book = await context.Books.FirstOrDefaultAsync(e => e.BookId == Guid.Parse("8bedf209-39fe-4bfa-8bec-86cf54925a76"));
    ShowBook(book);
}
```

出力は次のとおり。 `context.Books.FirstOrDefaultAsync(...)` なので `DbBook`
のデータしか読み込まれていない。そのため `PublisherName` は取得しておらず、
`null` となっている。

```
// DBからDbBookを1件取得
BookId        :8bedf209-39fe-4bfa-8bec-86cf54925a76
Title         :達人プログラマー
PublisherId   :10cf6487-563a-4e54-9a88-5eeb55623faa
PublisherName :(NULL)
```

### PublisherつきでDbBookを1件取得

関連データを読み込むには `Include()` を使う。コードは以下のようになる。

```cs
private static async Task GetOneBookIncludePublisher()
{
    Console.WriteLine("* DBからPublisherつきでDbBookを1件取得");
    using var context = new DatabaseContext();
    var book = await context.Books.Include(e => e.Publisher)
                    .FirstOrDefaultAsync(e => e.BookId == Guid.Parse("8bedf209-39fe-4bfa-8bec-86cf54925a76"));
    ShowBook(book);
}
```

出力は以下。 `PublisherName` が取得できるようになったことがわかる。

```
// DBからPublisherつきでDbBookを1件取得
BookId        :8bedf209-39fe-4bfa-8bec-86cf54925a76
Title         :達人プログラマー
PublisherId   :10cf6487-563a-4e54-9a88-5eeb55623faa
PublisherName :オーム社
```

### IncludeなしでPublisherNameが取得できる場合

ここからがEFCoreの追跡の出番となる。まずDBから `DbPublisher`
を1件取得する。そのあとContextを閉じないままその `DbPublisher`
を関連データとして持つ `DbBook` を取得する。

```cs
private static async Task GetOneBookAfterGetPublisher()
{
    Console.WriteLine("* DBからまずDbPublisherを取得し、そのあとそのDbPublisherに紐づくDbBookを取得");
    using var context = new DatabaseContext();

    _ = await context.Publishers.FirstOrDefaultAsync(e => e.PublisherId == Guid.Parse("10cf6487-563a-4e54-9a88-5eeb55623faa"));
    var book = await context.Books.FirstOrDefaultAsync(e => e.BookId == Guid.Parse("8bedf209-39fe-4bfa-8bec-86cf54925a76"));
    ShowBook(book);
}
```

以下の結果のとおり、明示的に `Include()` しなくても関連データとして `Publisher`
が読み込まれていることがわかる。

```
// DBからまずDbPublisherを取得し、そのあとそのDbPublisherに紐づくDbBookを取得
BookId        :8bedf209-39fe-4bfa-8bec-86cf54925a76
Title         :達人プログラマー
PublisherId   :10cf6487-563a-4e54-9a88-5eeb55623faa
PublisherName :オーム社
```

では他の `DbBook` に対してはどうなるのか？次のコードはDBから1件 `DbPublisher`
を取得し、そのあとContextを閉じないまますべての `DbBook` を取得している。

```cs
private static async Task GetAllBooksAfterGetOnePublisher()
{
    Console.WriteLine("* DBからまずDbPublisherを1件取得し、そのあとそのDbBookをすべて取得");
    using var context = new DatabaseContext();

    _ = await context.Publishers.FirstOrDefaultAsync(e => e.PublisherId == Guid.Parse("10cf6487-563a-4e54-9a88-5eeb55623faa"));
    var books = await context.Books.ToArrayAsync();
    foreach (var book in books)
    {
        ShowBook(book);
    }
}
```

以下のように事前に取得しておいた `DbPublisher` を関連データとして持つ `DbBook`
のみ、 `PublisherName` が読み込まれていることがわかる。EFCoreがデータを追跡し、
`PublisherId` を介して `DbBook` の関連データを自動で埋めているからだ。

```
// DBからまずDbPublisherを1件取得し、そのあとそのDbBookをすべて取得
BookId        :6b83ccf1-ffe7-4153-9816-cb009e710009
Title         :暗号技術のすべて
PublisherId   :acb75290-ef7d-44f8-b271-daebe35f62ee
PublisherName :(NULL)
BookId        :8bedf209-39fe-4bfa-8bec-86cf54925a76
Title         :達人プログラマー
PublisherId   :10cf6487-563a-4e54-9a88-5eeb55623faa
PublisherName :オーム社
BookId        :96b1c2fc-1372-4359-a0f1-bdff9c6443f4
Title         :正規表現技術入門
PublisherId   :c872a122-73f8-4720-8d08-65814dc3ecfc
PublisherName :(NULL)
```

### 追跡なしで読み込みたい場合

追跡は便利なときもあるが、自動で予期せぬプロパティをsetされると困る場合もある。そういった可能性がある場合は
`AsNoTracking()` を使って追跡を明示的に無効化する。

```cs
private static async Task GetOneBookWithoutTrackingAfterGetPublisher()
{
    Console.WriteLine("* DBからまずDbPublisherを取得し、そのあとそのDbPublisherに紐づくDbBookをAsNoTrackingで取得");
    using var context = new DatabaseContext();

    _ = await context.Publishers.FirstOrDefaultAsync(e => e.PublisherId == Guid.Parse("10cf6487-563a-4e54-9a88-5eeb55623faa"));
    var book = await context.Books.AsNoTracking().FirstOrDefaultAsync(e => e.BookId == Guid.Parse("8bedf209-39fe-4bfa-8bec-86cf54925a76"));
    ShowBook(book);
}
```

以下のとおり関連データの `Publisher` が取得されなくなったことがわかる。

```
// DBからまずDbPublisherを取得し、そのあとそのDbPublisherに紐づくDbBookをAsNoTrackingで取得
BookId        :8bedf209-39fe-4bfa-8bec-86cf54925a76
Title         :達人プログラマー
PublisherId   :10cf6487-563a-4e54-9a88-5eeb55623faa
PublisherName :(NULL)
```

### （番外編）Includeはいつ必要なのか

関連データを読み込む場合は `Include()`
を使う。この「読み込む」とは結果として取得することだけを意味する。すなわち絞り込みの条件として使うだけであれば
`Include()` は必要ない。

以下のコードと結果から、 `Include(e => e.Publisher)` がなくとも
`FirstOrDefaultAsync(...)` で `e.Publisher.Name`
を条件にして絞り込みができていることがわかる。また、この `DbPublisher`
は追跡の対象にもなっていない（関連データとして読み込まれていない）。

```cs
private static async Task GetOneBookByPublisher()
{
    Console.WriteLine("* 番外編 : Publisher.Nameを経由してBookを取得");
    using var context = new DatabaseContext();

    var book = await context.Books.FirstOrDefaultAsync(e => e.Publisher.Name == "翔泳社");
    ShowBook(book);
}
```

```
// 番外編 : Publisher.Nameを経由してBookを取得
BookId        :6b83ccf1-ffe7-4153-9816-cb009e710009
Title         :暗号技術のすべて
PublisherId   :acb75290-ef7d-44f8-b271-daebe35f62ee
PublisherName :(NULL)
```

## まとめ

EFCoreは読み込んだデータを自動で追跡する。読み込んだデータとはプログラム上で明示的に取得したレコードのことである。また、追跡は単に変更を検出するだけではなく、主キーをもとに関連データの紐づけを行うことも含む。

開発者としてこの挙動をどう扱うべきか？
基本的に追跡による関連データの読み込みに頼るべきではない。事前の処理によって異なる結果を返すクエリは間違いなくバグの元となる。むしろ
`null`
になっていると予想していたプロパティを追跡がsetしたことでトラブルが起きることもありそうだ。

役立ちうる場面としては、多数のデータを関連データ込みで取得したいが、関連データの種類が非常に少ない……などだろうか。問い合わせを2回に分けることでトータルのコストを減らせるかもしれない。ただしこれについては一切確認していない。
