# Osysharp.KnowledgeBase

A knowledge base and the help centre it is read in. Your team writes answers in markdown, a second person reviews them
before anyone else reads them, and each answer can have a version in every language you speak. Readers search, browse
by category, and get the version in their language, or the original when there is none yet. Answers meant only for
your team stay with your team, everywhere.

**Use case.** A photo editor's documentation. Writers draft "Open RAW files from your camera"; an editor publishes it.
A Swedish reader gets the Swedish version and an Arabic reader the Arabic one. Someone typing "my raw files show a grey
square" into the contact form sees that article offered before they send. The support team's refund playbook is in the
same knowledge base, and no reader ever finds it. The same kit runs a support desk's help centre, a product's docs or a
school's handbook: it never knows what the answers are about.

```osy
use Osysharp.KnowledgeBase@1;

policy ReadsTeamKnowledge => IsStaff;     // reads the articles only the team may read
policy WritesKnowledge => IsStaff;        // writes drafts and sends them for review
policy ReviewsKnowledge => IsEditor;      // publishes, sends back, archives
policy ManagesKnowledge => IsAdmin;       // categories, sections and settings

app.Knowledge = new KnowledgeSetup { Name = "Lumen Docs", ContactPath = "/contact" };

// a contact form whose subject offers articles as it is typed
KnowledgeSuggestingField("What is it about?", value: subject);

// a search small enough for a chat panel
KnowledgeQuickSearch();
```

The help centre is at `/help-centre`, the team's desk at `/knowledge`, and the settings at `/settings/knowledge`. The
desk's pages stand in your app's own shell — design `WorkspaceShell` once and they, and your settings, are drawn inside
it — and the knowledge base puts itself in your navigation (`Nav.Pages()` lists it, for the people who may write).
Design `HelpCentreFrame` once to put the whole help centre inside your site's header and footer. It is the help
centre's layout, so it stays on screen as a reader moves between pages: a search, a chat or an account in it keeps
what it holds.

## Your provider, your key

Nothing is provided, and nothing needs one. Search by words works out of the box. Search by meaning
(`KnowledgeByMeaning`) uses the embedding provider **you** choose for your app (`app.Embedding`), with **your** key as a
secret; until it has one, search keeps working by words and says so.

## What it does

| | |
|---|---|
| **Markdown articles** | written in the markdown editor, in categories and sections; a title, a line about it, and the words |
| **Drafts and versions** | one draft at a time per language; each publish is a numbered version, and an earlier one can be brought back as a new draft |
| **Review before publishing** | a reviewer who did not write a version publishes it, or sends it back saying what to change. Turn review off and a reviewer may publish their own. Each step is an event your `[On]` handler hears — `ArticleSentForReview`, `ArticleSentBack`, `ArticlePublished` — and each says itself in a bell or on a timeline ("Ada Lindqvist sent an article for review.") |
| **Nothing published by an edit** | moving a version is a step somebody takes, checked by the kit after it is saved, so a draft cannot be published by changing it |
| **Every language** | each language has its own versions. A new one starts as a copy of the original to translate; a reader with no version in their language gets the original, and the page says so |
| **For the team only** | an article, or a whole category, can be the team's. It never reaches a reader outside it: not on a page, not in a search, not in a suggestion |
| **Search** | full text over every published version, every word first and any word when nothing has them all, each article once and in the reader's language. An inflected word finds its other forms in Swedish, Danish, Norwegian, English and Arabic ("betalningen" finds "betalningar"). With an embedder, `KnowledgeByMeaning` finds the article that answers a question asked in other words, or in another language |
| **Suggestions as people type** | under a contact form's subject. An article opens over the form, so what they wrote is still there |
| **Search engines** | each page's title, description and canonical address, a `TechArticle` and its breadcrumb, every language's address, `noindex` for the team's own, and a sitemap |
| **Archive** | an archived article is read by nobody outside the team and is in no search; bring it back any time |
| **Your look** | the front page, a category, an article and the search are parts a site's look can design |
| **Personal data** | who wrote and reviewed what is classified. With `Osysharp.Privacy`, add `Osysharp.Privacy.KnowledgeBase`: a writer's name comes off their articles when they ask, and the articles stay |

## Screens

| | |
|---|---|
| The front page | ![Search, categories and recently updated](docs/front-wide-light.png) |
| An article | ![An article with more from its section](docs/article-wide-light.png) |
| In Arabic, on a phone | ![The same article, right to left](docs/article-ar-phone-light.png) |
| A suggestion opened over the contact form | ![The article over the form](docs/suggestion-open-wide-light.png) |
| Writing | ![The draft in the editor, where it sits and its history](docs/desk-editor-wide-light.png) |
| Reviewing | ![A version in review, with publish and send back](docs/review-article-wide-light.png) |

## Its tests

Every behaviour above is proven against a small documentation site, and its screens are photographed with
`osy test --pixels`. Each promise about who reads what is checked to fail when its rule is taken out.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.KnowledgeBase;
using KnowledgeKitTests;

[Test(Lumen)]
void a_category_with_nothing_published_is_not_on_the_front_page() {
  Published("installing", "Install Lumen on a Mac", "Drag it to Applications.");
  runas (Pia) {
    var shelves = KnowledgeShelves();
    Assert.Equal(["getting-started"], shelves.Select(s => s.Slug).ToList());
    Assert.Equal(1, shelves[0].Count);
    // …and an empty section is not on its category's page either; nor is an empty category's page anything.
    Assert.Equal(["installing"], KnowledgeShelf("getting-started")!.Sections.Select(s => s.Slug).ToList());
    Assert.Null(KnowledgeShelf("editing-photos"));
  }
}
```

From [`tests/audience.test.osy`](tests/audience.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.KnowledgeBase;
using KnowledgeKitTests;

[Test(Lumen)]
void the_front_page_counts_every_category_at_once() {
  Published("installing", "Install Lumen", "Download it, open it.");
  Published("colour", "White balance", "Pick something grey.");
  var small = 0;
  var shown = 0;
  runas (Pia) {
    var mark = Cost.Mark();
    shown = KnowledgeShelves().Count;
    small = Cost.StatementsSince(mark);
  }
  for (var i = 1; i <= 4; i++) {
    var section = "";
    runas (Owen) {
      var sid = AddKnowledgeSection(AddKnowledgeCategory("Topic " + i), "Basics " + i);
      section = KnowledgeSection.Single(s => s.Id == sid).Slug;
    }
    Published(section, "Article " + i, "Words " + i + ".");
  }
  runas (Pia) {
    var mark = Cost.Mark();
    var shelves = KnowledgeShelves();
    var grown = Cost.StatementsSince(mark);
    Log.Information("KnowledgeShelves: " + small + " statements with " + shown + " categories, " + grown + " with " + shelves.Count);
    Assert.Equal(shown + 4, shelves.Count);
    Assert.Equal(1, shelves.Last().Count);
    Assert.Equal(small, grown);
  }
}
```

From [`tests/cost.test.osy`](tests/cost.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.KnowledgeBase;
using KnowledgeKitTests;

[Test(Lumen)]
void a_reader_reads_their_language_and_falls_back_to_the_original() {
  var slug = Published("colour", "White balance", "Pick a neutral grey with the dropper.", summary: "Fix a colour cast.");
  Translated(slug, "sv", "Vitbalans", "Välj en neutral grå med pipetten.", summary: "Rätta till ett färgstick.");
  // Swedish: the Swedish version.
  Session.SetLanguage("sv");
  var sv = KnowledgeArticleView(slug);
  Assert.Equal("Vitbalans", sv.Title);
  Assert.Equal("sv", sv.Language);
  Assert.False(sv.InAnotherLanguage);
  Assert.Equal(["en", "sv"], sv.Languages);
  // Arabic: no Arabic version yet — the original, and the page says so.
  Session.SetLanguage("ar");
  var ar = KnowledgeArticleView(slug);
  Assert.Equal("White balance", ar.Title);
  Assert.Equal("en", ar.Language);
  Assert.Equal("ar", ar.ReaderLanguage);
  Assert.True(ar.InAnotherLanguage);
}
```

From [`tests/languages.test.osy`](tests/languages.test.osy).

## Learn more

- [The Osysharp.KnowledgeBase reference](https://osysharp.com/reference/ui/knowledge-kit/)
- [Osysharp.KnowledgeBase in the package catalogue](https://osysharp.com/packages/#osysharp-knowledgebase)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
