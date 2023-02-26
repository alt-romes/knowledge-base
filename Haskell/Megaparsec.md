Official tutorial: https://markkarpov.com/tutorial/megaparsec.html

Some simple example code for setting up megaparsec for users already familiar but that have forgotten the details

Create a parser type with `Void` (for errors) and `Text` (for inputs) over `Identity`.

```haskell
type Parser = ParsecT Void Text Identity
```

> Parsing of white space is an important part of any parser. We propose a convention where **every lexeme parser assumes no spaces before the lexeme and consumes all spaces after the lexeme**

To define combinators that take whitespace, define a space consuming function and wrap calls from the lexer module such as `lexeme` and `symbol`. For example:

```haskell
import Text.Megaparsec
import Text.Megaparsec.Char
import qualified Text.Megaparsec.Char.Lexer as L

-- space consumer
sc :: Parser ()
sc = L.space
  space1
  (L.skipLineComment "//")
  (empty)

lexeme :: Parser a -> Parser a
lexeme = L.lexeme sc

symbol :: Text -> Parser Text
symbol = L.symbol sc
```

And then, we could define `parens` and `integer` consuming trailing space as:

```haskell
parens :: Parser a -> Parser a
parens = between (symbol "(") (symbol ")")

integer :: Parser Integer
integer = lexeme L.integer
```

Running a parser:

```haskell

```

## Examples

A parser for `[//[user:password@]host[:port]]` ([source](https://markkarpov.com/tutorial/megaparsec.html#controlling-backtracking-with-try)):

```haskell
pUri :: Parser Uri
pUri = do
  uriScheme <- pScheme
  void (char ':')
  uriAuthority <- optional . try $ do            -- (1)
    void (string "//")
    authUser <- optional . try $ do              -- (2)
      user <- T.pack <$> some alphaNumChar       -- (3)
      void (char ':')
      password <- T.pack <$> some alphaNumChar
      void (char '@')
      return (user, password)
    authHost <- T.pack <$> some (alphaNumChar <|> char '.')
    authPort <- optional (char ':' *> L.decimal) -- (4)
    return Authority {..}                        -- (5)
  return Uri {..}                                -- (6)
```

A parser that consumes any printable character (letters, spaces, punctuation,...) between square brackets

```haskell

```

For a complete and interesting example see [parsing a simple imperative language](https://github.com/mrkkrp/megaparsec-site/blob/master/tutorials/parsing-simple-imperative-language.md).


