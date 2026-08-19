# markex

`markex` is a fast, non-validating extractor for structured markup embedded in text. It finds configured elements without building a complete document tree, making it useful when an application needs known payloads such as file directives, tool calls, or metadata blocks.

The primary API is [`tag`]. It supports XML-compatible tags by default, configurable delimiter fences, owned results, and zero-copy borrowed results.

## Quick start

Extract XML-compatible elements and preserve the text that surrounds them:

```rust
use markex::tag::{self, Part, TagOptions};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let input = "Before <DATA id=123>payload</DATA> after.";
    let parts = tag::extract(input, &["DATA"], TagOptions::default().with_capture_text(true));

    for part in parts {
        match part {
            Part::Text(text) => println!("Text: {text:?}"),
            Part::TagElem(element) => {
                println!("Tag: {}", element.tag);
                println!("Content: {}", element.content);
            }
        }
    }

    Ok(())
}
```

[`tag::TagOptions::with_capture_text`] controls whether text outside matched elements is emitted as `Part::Text`. Text capture is disabled by default.

## Custom fences

Use a [`tag::TagFence`] when the structured payload uses delimiters other than XML. `markex` includes [`tag::FENCE_XML`] and [`tag::FENCE_BRACKETS`], and applications can define their own fence values. A fence can also accept alternate closing delimiters through [`tag::TagFence::close_delim_alts`].

```rust
use markex::tag::{self, FENCE_BRACKETS, TagOptions};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let input = r#"[[[BIG_CONTENT path="/some/path.txt"]]]
... some big content
[[[/BIG_CONTENT]]]"#;
    let parts = tag::extract(
        input,
        &["BIG_CONTENT"],
        TagOptions::default().with_fence(FENCE_BRACKETS),
    );
    let file = parts
        .into_tag_elems()
        .into_iter()
        .next()
        .ok_or("expected a BIG_CONTENT element")?;

    assert_eq!(file.tag, "BIG_CONTENT");
    assert_eq!(file.content, "\n... some big content\n");

    Ok(())
}
```

A custom fence defines the opening delimiter, closing delimiter, and prefix that identifies closing tags:

```rust
use markex::tag::{self, TagFence};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let fence = TagFence {
        name: "mustache",
        open_delim: "{{",
        close_delim: "}}",
    close_delim_alts: None,
    closing_tag_prefix: "/",
    self_closing_suffix: "/",
    };
    let parts = tag::extract(
        "{{DATA}}value{{/DATA}}",
        &["DATA"],
        TagOptions::default().with_fence(fence),
    );
    let element = parts
        .into_tag_elems()
        .into_iter()
        .next()
        .ok_or("expected a DATA element")?;

    assert_eq!(element.content, "value");

    Ok(())
}
```

`FENCE_BRACKETS` accepts its canonical `]]]` delimiter and the tolerant `]]` alternate. When both delimiters begin at the same position, the canonical, longer delimiter is selected. Keeping bracket tags on separate lines from large payloads can help LLMs generate structured output without confusing the tag syntax with content.

## Owned and borrowed extraction

[`tag::extract`] returns owned [`tag::Parts`] values. Use [`tag::extract_refs`] when the input must remain available and allocations for extracted strings should be avoided. These return [`tag::PartsRef`], whose text, tag names, attributes, and content borrow from the input. Configure custom fences and text capture with [`tag::TagOptions`].

For streaming processing, use [`tag::TagIter`] or [`tag::TagRefIter`] directly.
