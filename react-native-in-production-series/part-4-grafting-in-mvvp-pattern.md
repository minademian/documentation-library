# Part IV - Grafting in the MVVP Pattern Into a Legacy React Native Codebase

_This is the fourth installment in this series entitled React Native App in Production Series, on maintaining "_Koptiska liturgiska texter_" (KLT, "Coptic liturgical prayers" in English), a digital repository for liturgical (communal) prayers in Arabic and Swedish. Read [part I](part-1-lessons-learned.md), [part II](part-2-grafting-in-TDD.md), and [part III](part-3-using-adapter-pattern.md) here._

## Table of Contents

- [Some Background and the Problem](#some-background-and-the-problem)
- [The Solution: MVVP Architecture](#the-solution-mvvp-architecture)
- [Why This Architecture](#why-this-architecture)
- [Implementation Strategy](#implementation-strategy)
- [Comparison with Original Architecture](#comparison-with-original-architecture)
- [Alternatives Considered](#alternatives-considered)
- [Limitations and When Not to Use This](#limitations-and-when-not-to-use-this)
- [Takeaways for Engineers](#takeaways-for-engineers)
- [What's Next](#whats-next)

## Some Background and the Problem

There have been requests and desires from the local church community over the past couple of years to add new liturgical texts to the app. However, the existing architecture of the app made it difficult to add new content without significant refactoring. 

The original codebase followed a **database-first approach**, where screens queried SQLite directly, looped through result sets, and rendered whatever came back. The structure of the content was implicit in the database schema. Language handling was scattered throughout the codebase with ternaries and conditionals.

```javascript
// The old way: database-first
db.executeSql("SELECT * FROM sentences WHERE pray_id = ?", [prayId]);

for (let i = 0; i < results.rows.length; i++) {
  const text = lang === "ar" ? row.arabic : row.swedish;
  sentences.push(text);
}
```

### Limitations With This Approach

- Structure is invisible without querying the database.
- Difficult to modify order or add conditional sections.
- Language handling is scattered across files.
- No separation between data fetching, business logic, and presentation.
- Hard to test without a live database.

When it seemed prudent and the codebase had achieved some stability, it was the right time to add a new liturgical text to the app. This presented an opportunity to introduce a new architecture for content management that wouldn't get in the way of the existing codebase. Also, it would serve as a proof of concept for the future. It could be used to migrate the rest of the app and to implement new features.

I saw an opportunity to introduce a new architecture for content management. Rather than replicating the existing pattern, I decided to implement a layout-first [MVVP (Model-View-ViewModel-Presenter)](https://www.linkedin.com/pulse/mastering-mvvp-unveiling-essence-modern-software-arup-kumar-nag--s1hvc/) architecture. MVVP extends MVVM by introducing a Presenter layer that separates business transformation from presentation formatting. In standard MVVM, the ViewModel handles both. Here, the ViewModel (hook) orchestrates, while two Presenter modules handle domain hydration and component formatting respectively.

## The Solution: MVVP Architecture

The goal was to make the content structure explicit and decouple the layers, so each could be tested and modified independently.

The new liturgical text to be implemented was the wedding ceremony - the crowning rite in Orthodox parlance. This is a complex ceremony with multiple sections, conditional content, and multilingual text. It also introduces the idea of liturgical rubrics.

> Rubrics have two primary functions in liturgical texts. They explain what the priest is doing. And sometimes, it may provide instructions for the congregation to be done during the service.

The architecture separates concerns into four layers:

```
screens/crowning/
├── crowning-rite.js           # View (Container)
├── hooks/
│   └── useRiteContent.js      # ViewModel (Hook)
├── views/
│   ├── rite.business.js       # Presenter (Business Logic)
│   └── rite.presentation.js   # Presenter (Formatting)
└── text/
    └── rite.json              # Model (Data)
```

### Layer Responsibilities

**Model (text/rite.json)**: Declarative content structure. The layout is visible and version-controlled. No database queries needed.

```json
{
  "text_id": 1,
  "text_slug": "Crowning Rite",
  "headings": { "ar": "سر الزيجة", "sv": "Vigelse" },
  "sections": [
    {
      "section_type": "section_header",
      "title": { "ar": "صلاة قبل الاعتراف", "sv": "Inledningsbön" }
    },
    {
      "section_type": "content_entry",
      "content": { "ar": "...", "sv": "..." }
    }
  ]
}
```

**ViewModel (useRiteContent hook)**: Orchestrates data fetching and transformation. The hook imports business and presentation logic, combines them, and returns formatted content.

```javascript
export function useRiteContent() {
  const [content, setContent] = useState(null);

  useEffect(() => {
    const hydratedContent = buildRiteContent();
    const componentData = formatForComponent(hydratedContent);
    setContent(componentData);
  }, []);

  return content;
}
```

**Presenter (business + presentation)**: Two-stage transformation. Business logic hydrates raw JSON into domain objects. Presentation logic formats those objects for the component.

```javascript
// Business layer: domain objects
hydratedSections.push({
  type: "content_entry",
  content: new MultilingualText({
    arabic: section.content.ar,
    coptic: section.content.cop,
    swedish: section.content.sv,
  }),
});

// Presentation layer: component-ready format
return {
  type: section.type,
  content: section.content.getAll(),
};
```

**View (crowning-rite.js)**: The screen component. Receives formatted content from the hook and delegates rendering. Minimal logic.

```javascript
const CrowningRite = ({ navigation }) => {
  const content = useRiteContent();
  const { currentLocale } = useSettings();

  if (!content) return null;

  return (
    <View>
      <Header title={content.screenMetadata.title[currentLocale]} />
      <ThreeColumn content={content.sections} />
    </View>
  );
};
```

### The MultilingualText Class

A key piece of the architecture is centralizing language handling. Instead of the existing architecture's ternaries scattered throughout the codebase, all multilingual content flows through a single class:

```javascript
export class MultilingualText {
  constructor(data) {
    this.ar = data.arabic || "";
    this.sv = data.swedish || "";
    this.cop = data.coptic || null;
  }

  get(lang) {
    const langMap = { arabic: this.ar, swedish: this.sv, coptic: this.cop };
    return langMap[lang] || this.ar;
  }

  getAll() {
    return { arabic: this.ar, swedish: this.sv, coptic: this.cop };
  }

  hasCoptic() {
    return this.cop !== null && this.cop !== "";
  }
}
```

This eliminates the pattern of `lang === "ar" ? sentence.arabic : sentence.swedish` that was duplicated across dozens of files.

> Coptic is a liturgical language used in the Coptic Orthodox Church. It is not commonly spoken anymore, but is preserved in religious texts and services. The MultilingualText class allows for the possibility of including Coptic text, allowing us to support it.

## Why This Architecture

### Maintainability

Changes are isolated to their layer:
- _Content structure changed?_ **Edit the JSON file**.
- _Language handling changed?_ **Edit MultilingualText**.
- _Display format changed?_ **Edit the presentation layer**.
- _Navigation changed?_ **Edit the screen component**.

### Discoverability

The content structure is visible in the JSON file. New contributors can understand what the screen displays without querying the database or tracing through code.

### Component Composition

The MVVP layers give you clean, well-structured data. But the View layer itself should also express intent rather than mechanics. In the old codebase, screens looped over database rows and rendered generically — the component didn't express _what_ it was showing, just that it was iterating.

The logical next step is to make the View layer declarative. Instead of flat component exports, components are attached as properties on namespace objects — compound components that read as declarations of intent:

```javascript
const Layout = {};

Layout.CrowningPrayer = function CrowningPrayer({ content, nightMode, onBack }) {
  // Presentational component: renders the crowning rite layout
  // ...
};

const FeatureScreen = {};

FeatureScreen.CrowningRite = function CrowningRite({ navigation }) {
  const content = useRiteContent();

  // ...loading and error states

  return (
    <View style={[styles.container, nightMode ? styles.nightModeBackground : styles.lightModeBackground]}>
      <Layout.CrowningPrayer
        content={content}
        nightMode={nightMode}
        onBack={handleBack}
      />
    </View>
  );
};
```

`<Layout.CrowningPrayer />` reads as "render the Crowning Prayer layout" — not "iterate over rows and figure out what to display." The screen component orchestrates; the layout component presents. And `Layout` can grow naturally to hold `Layout.ConfessionPrayers`, `Layout.RepentancePrayers`, and so on, without restructuring directories or inventing new conventions.

This compositional pattern — namespace objects as component containers — was adapted from [Fernando Rojo's keynote at React Universe 2025](https://youtu.be/4KvbVq3Eg5w?si=b7yRmkwlBcQwvUT3), where he demonstrated how compound components create self-documenting call sites in React.

## Implementation Strategy

I implemented this incrementally:

1. Create the JSON data file with the content structure
2. Build the MultilingualText class for language handling
3. Implement business logic to hydrate JSON into domain objects
4. Implement presentation logic to format for components
5. Create the hook to orchestrate the pipeline
6. Wire the screen to use the hook

The old database-first approach remains in the codebase for other screens. This new screen serves as a proof of concept for migrating the rest.

## Comparison with Original Architecture

| Aspect | Database-First | MVVP Layout-First |
|--------|---------------|-------------------|
| Content structure | Implicit in DB | Explicit in JSON |
| Language handling | Scattered ternaries | Centralized class |
| Testability | Requires database | Each layer testable |
| Data flow | Query → loop → render | JSON → hydrate → format → render |
| Discoverability | Low | High |

## Alternatives Considered

**Staying with database-first.** The path of least resistance was to add the Crowning Rite to SQLite like every other screen. But this would have meant replicating the same scattered language handling, implicit content structure, and untestable data flow. The whole point was to stop compounding that pattern. Adding one more screen the old way would have made the eventual migration harder, not easier.

**MVVM without the Presenter split.** A standard MVVM approach would put both hydration and formatting in the ViewModel hook. This works for simple screens, but the Crowning Rite has multilingual text, liturgical rubrics, and conditional sections. Separating business transformation (raw JSON to domain objects) from presentation transformation (domain objects to component props) keeps each stage focused and independently testable. Collapsing them into one layer would have obscured the boundary between "what does this content mean?" and "how should it render?"

**Headless CMS or external content service.** A CMS would externalize content management entirely. But KLT is an offline-first app — users open it during church services. Adding a network dependency for content that changes infrequently would introduce failure modes without meaningful benefit. JSON files bundled with the app are simpler, faster, and version-controlled.

## Limitations and When Not to Use This

**JSON doesn't scale like a database.** With one liturgical text, a single JSON file is easy to read and maintain. With fifty, you lose the queryability that SQLite provides — filtering across texts, searching by keyword, or aggregating metadata all require loading and parsing every file. If KLT grows to that point, a hybrid approach (JSON for layout structure, database for cross-text queries) may be necessary.

**Full content lives in memory.** The current approach loads the entire JSON file into memory and hydrates it into domain objects. For the Crowning Rite this is negligible, but liturgical texts with hundreds of sections could put pressure on lower-end devices. SQLite's cursor-based approach streams rows on demand, which is more memory-efficient for large datasets.

**Content changes require a release.** Because JSON files are bundled with the app, any content correction — even a typo — requires a new build and app store submission. This is acceptable for liturgical texts that change rarely, but this pattern would be a poor fit for content that needs to update independently of the app release cycle.

## Takeaways for Engineers

**Layout-first thinking inverts the control flow.** Instead of "what does the database give me?", the question becomes "what structure do I need, and how do I fill it?" This makes the content structure a first-class citizen.

**Separation of concerns pays dividends at the seams.** The hook knows about business and presentation logic, but not about JSON structure. The business logic knows about domain objects, but not about component props. Each layer has a single responsibility.

**Start with one screen.** Introducing a new architecture to a legacy codebase doesn't require a rewrite. Pick a new feature, implement it with the new pattern, and let it prove itself before migrating existing code.

**Centralize repeated patterns.** The MultilingualText class eliminated dozens of duplicated ternaries. Identifying and centralizing these patterns reduces bugs and makes the codebase more consistent.

## What's Next

The Crowning Rite implementation validates the pattern. Next steps include:
- Applying the same architecture to the Agpeya screens (8 hours of prayer)
- Generating JSON files from the existing database for migration
- Adding TypeScript types to the domain objects
- Implementing error boundaries at the container layer

_KLT is available on the [Play](https://play.google.com/store/apps/details?id=com.copticapps.copticprayersfree&hl=sv) and [App](https://apps.apple.com/se/app/koptiska-liturgiska-texter/id1441254651) Stores. If you're working through similar architectural challenges in legacy React Native codebases, I'd be interested in your perspective._