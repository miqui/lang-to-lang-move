# lang-to-lang-move

What to expect when developers move from one language to another.

Each section captures the friction points, missing guardrails, and practical
adaptations for a developer transitioning from one language to another.

## Sections

| Move | Guide |
|---|---|
| Java → Python | [java-to-python](java-to-python/) |
| Java → Go | [java-to-golang](java-to-golang/) |
| Java → Node.js | [java-to-nodejs](java-to-nodejs/) |

## Structure

Each language move lives in its own directory:

```
lang-to-lang-move/
├── README.md            ← this file (index of all moves)
└── <from>-to-<to>/      ← e.g. java-to-python
    └── README.md        ← the guide for that move
```

## Contributing

To add a new language move:

1. Create a directory named `<from>-to-<to>` (e.g. `java-to-kotlin`, `python-to-go`).
2. Write a `README.md` inside it covering the key adjustments: type systems,
   tooling, dependency management, testing, idioms, and common complaints.
3. Add the move to the index table above.
