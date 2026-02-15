# Phase 0 Formal Grammar (EBNF)

This is the **authoritative grammar** for Phase 0 PEPL. The parser MUST accept exactly this grammar. Illustrative PEPL in other documents may use simplified syntax for readability — this grammar is the source of truth.

**Key grammar constraints:**
- Statements are separated by newlines. Semicolons are NOT used. Each statement must be on its own line. Multiple statements on one line are a parse error.
- Block ordering is enforced: types → state → capabilities → credentials → derived → invariants → actions → views
- Tests live OUTSIDE the space declaration
- Only `//` single-line comments (no block comments)
- Lambdas use block body only (no expression body)
- No user-defined generics — only built-in parameterized types (`list<T>`, `Result<T, E>`)
- No `record<{}>` syntax — use `{ field: type }` anonymous record types
- String interpolation uses `${expr}` (dollar-brace)
- Record spread: `{ ...record, field: value }`
- `?` postfix operator unwraps Result (traps on Err)
- `??` infix nil-coalescing operator
- Trailing commas allowed everywhere (records, args, params)
- Component children use a second brace block: `Modal { props } { children }`
- Action references in props are deferred (not eagerly evaluated)
- `+` operator works on numbers only — use `${expr}` for string concatenation
- `for` iterates lists only — use `list.range()` for counted loops

```ebnf
(* ─── Top Level ─── *)

Program          = SpaceDecl { TestsBlock } ;

SpaceDecl        = "space" Identifier "{" SpaceBody "}" ;

SpaceBody        = { TypeDecl }
                   StateBlock
                   [ CapabilitiesBlock ]
                   [ CredentialsBlock ]
                   [ DerivedBlock ]
                   { InvariantDecl }
                   { ActionDecl }
                   { ViewDecl }
                   [ UpdateDecl ]
                   [ HandleEventDecl ] ;

(* ─── Type Declarations ─── *)

TypeDecl         = "type" Identifier "=" TypeBody ;

TypeBody         = "|" Variant { "|" Variant }                 (* sum type  *)
                 | Type ;                                      (* type alias *)

Variant          = Identifier [ "(" ParamList ")" ] ;

(* ─── State ─── *)

StateBlock       = "state" "{" { StateField } "}" ;

StateField       = Identifier ":" Type "=" Expression ;

(* ─── Capabilities ─── *)

CapabilitiesBlock = "capabilities" "{" [ RequiredCaps ] [ OptionalCaps ] "}" ;

RequiredCaps     = "required" ":" "[" { Identifier [ "," ] } "]" ;

OptionalCaps     = "optional" ":" "[" { Identifier [ "," ] } "]" ;

(* ─── Credentials ─── *)

CredentialsBlock = "credentials" "{" { CredentialField } "}" ;

CredentialField  = Identifier ":" Type ;
                 (* Each field declares a host-managed secret.
                    Type is always 'string'. Credential names become
                    read-only bindings within the space, injected by host at runtime. *)

(* ─── Derived State ─── *)

DerivedBlock     = "derived" "{" { DerivedField } "}" ;

DerivedField     = Identifier ":" Type "=" Expression ;
                 (* Recomputed after every action. Read-only — cannot be set.
                    May reference state fields and prior derived fields. *)

(* ─── Invariants ─── *)

InvariantDecl    = "invariant" Identifier "{" Expression "}" ;

(* ─── Actions ─── *)

ActionDecl       = "action" Identifier "(" [ ParamList ] ")" Block ;

ParamList        = Param { "," Param } [ "," ] ;

Param            = Identifier ":" Type ;

(* ─── Views ─── *)

ViewDecl         = "view" Identifier "(" [ ParamList ] ")" "->" "Surface" UIBlock ;

UIBlock          = "{" { UIElement } "}" ;

UIElement        = ComponentExpr
                 | LetBinding
                 | IfExpr
                 | ForExpr ;
                 (* Inside UIBlock, IfExpr and ForExpr bodies contain UIElements,
                    not Statements. The parser determines context from the
                    enclosing block type (UIBlock vs Block). *)

ComponentExpr    = UpperIdentifier "{" { PropAssign } "}" [ "{" { UIElement } "}" ] ;
                 (* Two forms:
                    1. Props only:    Text { value: "hi", }
                    2. Props + children: Modal { visible: v, } { Column { ... } }
                    Phase 0 components: Text, Button, TextInput, Column, Row,
                    Scroll, ScrollList, ProgressBar, Modal, Toast. Unknown names produce E402. *)

PropAssign       = Identifier ":" Expression [ "," ] ;
                 (* Trailing comma is optional (allowed for consistency, not required).
                    Expression may be an ActionRef for event handlers:
                    on_tap: action_name,          — nullary action reference
                    on_tap: action_name(arg),     — action with bound argument
                    on_change: fn(v) { action(v) } — explicit lambda callback
                    Action references are NOT eagerly evaluated — they construct
                    callbacks. The compiler distinguishes ActionRef from FunctionCall
                    in prop position by checking if the identifier names an action. *)

UpperIdentifier  = UpperLetter { Letter | Digit } ;

UpperLetter      = "A"..."Z" ;

(* ─── Control Flow ─── *)

IfExpr           = "if" Expression Block [ "else" ( Block | IfExpr ) ] ;

ForExpr          = "for" Identifier [ "," Identifier ] "in" Expression Block ;
                 (* First identifier = item, optional second = index (0-based). *)

Block            = "{" { Statement } "}" ;
                 (* The value of a Block is the value of its last
                    expression-statement, if any. Actions ignore block
                    values; lambdas and if/match use them as return values. *)

Statement        = SetStatement
                 | LetBinding
                 | IfExpr
                 | ForExpr
                 | MatchExpr
                 | FunctionCall
                 | ReturnStmt
                 | AssertStmt
                 | Expression ;
                 (* Expression-statement: a bare expression whose value
                    becomes the block's return value if it is the last
                    statement. Standalone match and bare expressions are
                    both valid statements. *)

SetStatement     = "set" Identifier { "." Identifier } "=" Expression ;
                 (* Supports nested field access: set record.field = value
                    `set record.field = value` is equivalent to
                    `set record = { ...record, field: value }`.
                    For deeper nesting: `set a.b.c = x` desugars to
                    `set a = { ...a, b: { ...a.b, c: x } }`.
                    Set statements execute sequentially within an action.
                    Each set immediately updates the field — subsequent
                    set statements see the updated value. *)

LetBinding       = "let" ( "_" | Identifier [ ":" Type ] ) "=" Expression ;
                 (* "let _ = expr" is a discard binding — evaluates expr,
                    discards the result. Used to acknowledge unused return
                    values (e.g., from capability calls returning Result). *)

ReturnStmt       = "return" ;
                 (* Exits action early. All prior set statements are applied.
                    No return value — actions do not return values. *)

AssertStmt       = "assert" Expression [ "," StringLiteral ] ;

(* ─── Expressions ─── *)

Expression       = OrExpr ;

OrExpr           = AndExpr { "or" AndExpr } ;

AndExpr          = NilCoalesceExpr { "and" NilCoalesceExpr } ;

NilCoalesceExpr  = CompExpr { "??" CompExpr } ;
                 (* a ?? b — returns a if not nil, else b *)

CompExpr         = AddExpr { CompOp AddExpr } ;

CompOp           = "==" | "!=" | "<" | ">" | "<=" | ">=" ;

AddExpr          = MulExpr { ( "+" | "-" ) MulExpr } ;

MulExpr          = UnaryExpr { ( "*" | "/" | "%" ) UnaryExpr } ;

UnaryExpr        = [ "not" | "-" ] PostfixExpr ;

PostfixExpr      = PrimaryExpr { "?" | "." Identifier | "." Identifier "(" [ ArgList ] ")" } ;
                 (* ? = Result unwrap (traps on Err, returns Ok value)
                    .field = field access
                    .method(args) = qualified call *)

PrimaryExpr      = NumberLiteral
                 | StringLiteral
                 | BoolLiteral
                 | "nil"
                 | Identifier
                 | FunctionCall
                 | ListLiteral
                 | RecordLiteral
                 | "(" Expression ")"
                 | IfExpr
                 | MatchExpr
                 | LambdaExpr ;

FunctionCall     = Identifier "(" [ ArgList ] ")" ;

ArgList          = Expression { "," Expression } [ "," ] ;

ListLiteral      = "[" [ Expression { "," Expression } [ "," ] ] "]" ;

RecordLiteral    = "{" [ RecordContent { "," RecordContent } [ "," ] ] "}" ;

RecordContent    = SpreadExpr
                 | RecordField ;

SpreadExpr       = "..." Expression ;
                 (* { ...existing_record, field: new_value } *)

RecordField      = FieldName ":" Expression ;
                 (* FieldName may be a keyword used contextually — see FieldName below. *)

LambdaExpr       = "fn" "(" [ ParamList ] ")" Block ;
                 (* Block body only — no expression-body lambdas.
                    The lambda's return value is the value of the last
                    expression in its Block body. Lambdas cannot use
                    `return` (that keyword is action-only).
                    Example: fn(item) { item.value > 0 }
                    Multi-statement: fn(x) { let y = x * 2; y + 1 } *)

MatchExpr        = "match" Expression "{" MatchArm { "," MatchArm } [ "," ] "}" ;

MatchArm         = Pattern "->" ( Expression | Block ) ;

Pattern          = Identifier [ "(" PatternBindings ")" ]      (* variant match *)
                 | "_" ;                                        (* wildcard      *)

PatternBindings  = Identifier { "," Identifier } ;

(* ─── Types ─── *)

Type             = "number"
                 | "string"
                 | "bool"
                 | "nil"
                 | "any"                                        (* stdlib-only: core.log, etc.  *)
                 | "color"                                      (* UI color values              *)
                 | "Surface"                                    (* view return type             *)
                 | "InputEvent"                                 (* game loop input events       *)
                 | "list" "<" Type ">"                          (* list<number>                 *)
                 | "{" { RecordTypeField } "}"                  (* { name: string, age: number }*)
                 | "Result" "<" Type "," Type ">"               (* Result<string, HttpError>    *)
                 | "(" [ TypeList ] ")" "->" Type               (* (number) -> bool             *)
                 | Identifier ;                                 (* user-defined sum type name   *)

TypeList         = Type { "," Type } [ "," ] ;

RecordTypeField  = FieldName [ "?" ] ":" Type [ "," ] ;
                 (* FieldName may be a keyword used contextually — see FieldName below.
                    Optional marker: spacing?: number means field may be omitted.
                    Omitted optional fields default to nil. *)

FieldName        = Identifier | Keyword ;
                 (* In field-name positions (record types, record literals, state fields,
                    derived fields, and set-path segments after "."), keywords like
                    `color`, `type`, `state` etc. are valid field names. *)

(* ─── Testing ─── *)

TestsBlock       = "tests" "{" { TestCase } "}" ;
                 (* Lives OUTSIDE the space declaration.
                    Tests can reference the space's state and actions. *)

TestCase         = "test" StringLiteral [ WithResponses ] Block ;

WithResponses    = "with_responses" "{" { ResponseMapping } "}" ;

ResponseMapping  = QualifiedName "(" [ ExpressionList ] ")" "->" Expression "," ;
                 (* Maps capability calls to predetermined return values for
                    deterministic offline testing. If a test calls a capability
                    function not declared in with_responses, the call returns
                    Err with error code "unmocked_call". If the test has no
                    with_responses block, ALL capability calls return
                    Err("unmocked_call").
                    Example:
                    with_responses {
                      http.get("https://api.example.com/data") -> Ok({ status: 200, body: "{}", headers: [] }),
                      storage.get("key") -> Ok("value"),
                    } *)

(* ─── Game Loop (Phase 0B+, grammar-defined now) ─── *)

UpdateDecl       = "update" "(" Identifier ":" "number" ")" Block ;

HandleEventDecl  = "handleEvent" "(" Identifier ":" "InputEvent" ")" Block ;

(* ─── InputEvent Type (built-in sum type for game loop) ─── *)

(* InputEvent is a built-in sum type:
   type InputEvent =
     | KeyDown(key: string)
     | KeyUp(key: string)
     | PointerDown(x: number, y: number)
     | PointerUp(x: number, y: number)
     | PointerMove(x: number, y: number)
   Key values: "ArrowUp", "ArrowDown", "ArrowLeft", "ArrowRight", "Space", "Enter",
               "a".."z", "0".."9". Platform maps physical inputs to these keys.
   On mobile: touch events map to Pointer events.
   On glasses: gaze dwell maps to PointerDown at gaze position. *)

(* ─── Comments (stripped during lexing, not in AST) ─── *)

Comment          = "//" { Character } Newline ;
                 (* Single-line only. No block comments. *)

(* ─── Literals ─── *)

NumberLiteral    = Digit { Digit } [ "." Digit { Digit } ] ;

StringLiteral    = '"' { StringChar | Interpolation } '"' ;

StringChar       = EscapeSequence
                 | (* any Unicode character except unescaped '"' and unescaped '$' before '{' *) ;

EscapeSequence   = "\\" ( '"' | '\\' | 'n' | 't' | 'r' | '$' ) ;
                 (* \" = literal quote, \\ = backslash, \n = newline,
                    \t = tab, \r = carriage return, \$ = literal dollar sign *)

Interpolation    = "$" "{" Expression "}" ;
                 (* Native syntax. The compiler lowers this to
                    string.concat + convert.to_string internally.
                    Example: "Hello ${name}" → string.concat("Hello ", convert.to_string(name))
                    Multi-part: "${a} ${b}" → string.concat(string.concat(
                      convert.to_string(a), " "), convert.to_string(b))
                    Nested string literals inside ${} are supported:
                    "Status: ${if done { "complete" } else { "pending" }}" *)

BoolLiteral      = "true" | "false" ;

Identifier       = Letter { Letter | Digit | "_" } ;

(* ─── Lexical ─── *)

Letter           = "a"..."z" | "A"..."Z" ;

Digit            = "0"..."9" ;

Newline          = (* line terminator: LF or CRLF *) ;

Character        = (* any Unicode character *) ;
```

## Operator Precedence (highest to lowest)

| Precedence | Operator(s) | Associativity |
|---|---|---|
| 1 (highest) | `.` (field access), `()` (call) | Left |
| 2 | `?` (Result unwrap) | Postfix |
| 3 | Unary `-`, `not` | Right |
| 4 | `*`, `/`, `%` | Left |
| 5 | `+`, `-` | Left |
| 6 | `==`, `!=`, `<`, `>`, `<=`, `>=` | None (no chaining) |
| 7 | `??` (nil-coalescing) | Left |
| 8 | `and` | Left |
| 9 (lowest) | `or` | Left |

## Reserved Keywords (Phase 0)

```
space  state  action  view  set  let  if  else  for  in  match  return
invariant  capabilities  required  optional  credentials  derived  tests  test  assert
fn  type  true  false  nil  not  and  or
number  string  bool  list  color
update  handleEvent  Result  Surface  InputEvent
core  math  record  time  convert  json  timer
http  storage  location  notifications  clipboard  share
```

> **Module names are reserved.** `let math = 5` is compile error E101: "Cannot shadow reserved identifier 'math'". `string`, `list`, and `color` serve dual roles: type names in type annotations, module prefixes in qualified calls (e.g., `string.length(s)`). The parser disambiguates by context — `Identifier "." Identifier "("` is always a module call; standalone usage in type position is always a type.

> **Contextual field names.** All 52 keywords are valid as record field names. In record types (`{ color: string }`), record literals (`{ color: "#f00" }`), state/derived field declarations, and `set` path segments after `.` (`set theme.color = "#fff"`), the parser accepts any keyword as a field name. Keywords remain reserved in all other positions (e.g., `let color = 5` is still an error).

## Naming Conventions

| Entity | Convention | Example |
|---|---|---|
| Space names | PascalCase | `WaterTracker`, `TodoList` |
| State fields | snake_case | `daily_goal`, `items` |
| Actions | snake_case | `add_item`, `toggle_done` |
| View functions | snake_case | `main`, `item_row` |
| Type names (sum types) | PascalCase | `Priority`, `Shape` |
| Variant constructors | PascalCase | `High`, `Circle` |
| Stdlib calls | module.snake_case | `math.round`, `list.append` |
