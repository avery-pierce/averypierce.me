---
layout: post
title:  "Software architecture study: mini cactpot pt 1"
date:   2020-07-12 10:36:00 -0500
categories: software-architecture swift
summary: Follow along in this architectural study of Mini Cactpot. In part 1, I explore the software design of a cactpot scratch-off card, and try to design an ergonomic API for getting, setting, and querying its cells.
---

[Mini cactpot](https://na.finalfantasyxiv.com/lodestone/playguide/contentsguide/goldsaucer/cactpot/) is a chance-based game that resembles a scratch-off ticket. The game consists of a 3x3 grid containing the digits 1 through 9, and each digit is used exactly once. The player chooses a horizontal, vertical, or diagonal row of 3 squares. The sum of these squares is used to determine the player's payout, according to a pay table which is shown to the player. However, at the start of the game only 1 random square is revealed to the player. The player may then choose 3 more squares to reveal before picking a row.

It is possible to compute the probability of winning a certain amount from a certain row, given a partially-revealed card. Armed with this information, a savvy player can always choose the row that maximizes their chances of winning the jackpot. In this study, I will create a software module written in Swift that can compute the average payout for each row in a partially-completed mini cactpot ticket. 

<div class="note">
<p>This is not a programming tutorial. The goal of this post is not to teach someone to write code, it's an exercise for experienced programmers to sharpen their skills by examining trade-offs in a program's software architecture.</p>
</div>

## Ticket

The most obvious component of the system is the cactpot ticket itself. The programmer should be able to create a mini cactpot ticket, and set the value of any cell in an expressive way. In addition, the programmer will want to query for all the cells in a given horizontal, vertical, or diagonal row without any fuss. But the ticket itself is not responsible for computing probabilities. It's only responsible for getting and setting state.

Let's examine how we could model this in Swift. There are 9 cells. Each cell is either empty, or contains a digit 1-9. My instinct is to model this as the type `[Int?]`. This model isn't a perfect representation of our data – the array will permit more than 9 entries, and it will permit entries outisde the range of valid inputs. To discourage misuse of this data, I'll make it a readonly attribute of the `Ticket` struct. I chose to model the mini cactpot ticket as a `struct` instead of a `class`, so the value will represent a snapshot of state.

```swift
public struct Ticket: Hashable, Equatable {
  public private(set) var cells: [Int?]

  init(cells: [Int?]) {
    self.cells = cells
  }
}
```

This alone is enough to completely model a cactpot card at any stage of the game. I could attempt to validate the input of the `init` method, but what would I do if the input was not valid? If I throw an error, then the programmer could only be call the initializer with `try`. If I return `nil`, the programmer would be stuck with unwrapping an optional for state that they would know not to be `nil`. I could call `fatalError`, but crashing the app over invalid input seems hostile. So I will trust the programmer to do the right thing, and only use the initilizer with sane input. This is not perfect, but it's better than the alternatives.

## Convenience

The next step is to add methods and properties that will make the ticket struct easier to work with. Adding a static property called `empty` can be used to return a cactpot ticket which is completely blank. This will be a good starting point for future programs.

```swift
public struct Ticket: Hashable, Equatable {
  public private(set) var cells: [Int?]

  init(cells: [Int?]) {
    self.cells = cells
  }

  static var empty: Ticket {
    return Ticket(cells: [
      nil, nil, nil, 
      nil, nil, nil, 
      nil, nil, nil
    ])
  }
}
```

Now the programmer can create a blank `Ticket` by calliing `Ticket.empty`. The next thing the programmer will want to do is mutate the ticket, and write data into a given cell. We could allow the programmer to write to the `cells` array directly, however writing to the "seventh cell" is not intuitive. When I think about this ticket conceptually, I think of the cells along the horizontal and vertical axes. "Bottom left" is easier to think about than "seventh cell". Ideally, I would like an interface like this: `myTicket[.bottom, .left] = 3`.

We can accomplish this with Swift subscripts, but we first have to define enums for the horizontal and vertical rows. Since these rows only make sense within the context of a mini cactpot ticket, I will define them within the `Ticket` struct.

```swift
public struct Ticket: Hashable, Equatable {
  /* ... */

  public enum VerticalRow: Hashable {
    case left
    case center
    case right
  }

  public enum HorizontalRow: Hashable {
    case top
    case center
    case bottom
  }
}
```

I am thinking ahead here: Not only will these enums be helpful in accessing individual cells of a ticket, but I expect they will come in handy when we need to query for entire rows for scoring.

Now our ticket state is stored internally as `[Int?]`. That means each combination of horizontal and vertical rows corresponds to a single index in the card. How do I determine what that index is? I think it would be helpful to write a function that returns an index for a given horizontal and vertical row.

Since this function does not rely on any state internal to the `Ticket`, we'll make this a static function.

```swift
public struct Ticket: Hashable, Equatable {
  /* ... */

  private static func indexOf(_ h: HorizontalRow, _ v: VerticalRow) -> Int {
    switch (h, v) {
    case (.top, .left): return 0
    case (.top, .center): return 1
    case (.top, .right): return 2
    case (.center, .left): return 3
    case (.center, .center): return 4
    case (.center, .right): return 5
    case (.bottom, .left): return 6
    case (.bottom, .center): return 7
    case (.bottom, .right): return 8
    }
  }
}
```

I'm quite proud of how boring this function is. Some folks might be tempted to do something more clever – perhaps assign a value to the horiztontal axis, and a value to the vertical axis, then compute the sum of those values to get the index. However, my solution is straightforward. It may not scale well to `N` sized cactpot cards, but the scope of our problem is pretty narrow.

Let's move forward. With our coordinate space enumerated along two axes, and a function to translate a coordinate to a cell index, we can write our subscript functions with little pain.

```swift
public struct Ticket: Hashable, Equatable {
  /* ... */

  /// Access the cell at the given horizontal and vertical row
  public subscript(h: HorizontalRow, v: VerticalRow) -> Int? {
    get {
      let index = Ticket.indexOf(h, v)
      return cells[index]
    } set {
      let index = Ticket.indexOf(h, v)
      cells[index] = newValue
    }
  }
}
```

The end result of this work is a very convenient API for reading and writing cells in a mini cactpot ticket. Let's see it in action. Suppose you want to fill a ticket in this way:

```
+---+---+---+
| 7 |   |   |
+---+---+---+
|   | 5 |   |
+---+---+---+
|   | 1 | 2 |
+---+---+---+
```

The code to write that is plain and easy to follow:

```swift
var emptyTicket = Ticket.empty

emptyTicket[.top, .left] = 7
emptyTicket[.bottom, .center] = 1
emptyTicket[.center, .center] = 5
emptyTicket[.bottom, .right] = 2
```

Furthermore, accessing each cell is just as easy:

```swift
emptyTicket[.bottom, .center]
// Returns 1

emptyTicket[.top, .right]
// Returns nil
```

## Query scorable rows

Next we'll want to be able to compute the sum for any scorable row – that is, any horizontal, vertial, or diagonal row. There are exactly 8 possible rows to pick from. The natural way to model this is with an `enum`. I could create an enum with 8 explicit choices, but we run into a naming conflict with the word `center` – there is both a horizontal and vertical center. This could be worked around by naming these options `horizontalCenter` and `verticalCenter`, but I think I have a better solution. We have already enumerated the horizontal and vertical rows, so I would like to model it like so:

```swift
public struct Ticket: Hashable, Equatable {
  /* ... */

  public enum Row: Hashable {
    case vertical(VerticalRow)
    case horizontal(HorizontalRow)

    /**
      Includes the bottom-left and top-right cells

          +---+---+---+
          |   |   | X |
          +---+---+---+
          |   | X |   |
          +---+---+---+
          | X |   |   |
          +---+---+---+
    */
    case diagonalUpward
    
    /**
      Includes the top-left and bottom-right cells

          +---+---+---+
          | X |   |   |
          +---+---+---+
          |   | X |   |
          +---+---+---+
          |   |   | X |
          +---+---+---+
    */
    case diagonalDownward
  }
}
```

For the two diagonal rows, I chose `diagonalDownward` and `diagonalUpward`. These rows are tricky to clearly differentiate, and the name I landed on assumes the programmer thinks in a left-to-right language. Nevertheless, I've included clear documentation with a handy diagram to further disabiguate the two cases.

Now, our ticket needs a function that returns a collectioon of `Int?` for each possible row. I'm choosing to use an array `[Int?]`. The cells will be ordered left-to-right for horizontal and diagonal rows, and top-to-bottom to for vertical rows. I don't anticipate that the order will be terribly useful. I considered using a `Set<Int?>`, but there's a good chance our row will contain more than 1 `nil` value in the row, and the `Set` doesn't model that very well.

In order to return the `[Int?]` for the selected row, two things need to happen: First, we need to get a set of cell coordinates from the row. Next, we need to map those coordinates to an array of values.

Let's add a computed property to the `Row` enum above, to return the cell coordiantes for each cell in the row.

```swift
public struct Ticket: Hashable, Equatable {
  /* ... */

  public enum Row: Hashable {
    /* ... */

    public var coordinates: [(h: HorizontalRow, v: VerticalRow)] {
      switch self {
      case .vertical(let v):
        return [(.top, v), (.center, v), (.bottom, v)]
      case .horizontal(let h):
        return [(h, .left), (h, .center), (h, .right)]
      case .diagonalUpward:
        return [(.bottom, .left), (.center, .center), (.top, .right)]
      case .diagonalDownward:
        return [(.top, .left), (.center, .center), (.bottom, .right)]
      }
    }
  }
}
```

With this new tool, we can easily write a map function to rerturn the cells in the ticket.

```swift
public struct Ticket: Hashable, Equatable {
  /* ... */

  public func cells(in row: Row) -> [Int?] {
    return row.coordinates.map({ self[$0.h, $0.v] })
  }
}
```

## Bringing it together

Building on the sample code from before, I will demonstrate the new functionality for selecting whole rows 

```swift
var emptyTicket = Ticket.empty

emptyTicket[.top, .left] = 7
emptyTicket[.bottom, .center] = 1
emptyTicket[.center, .center] = 5
emptyTicket[.bottom, .right] = 2

/*
+---+---+---+
| 7 |   |   |
+---+---+---+
|   | 5 |   |
+---+---+---+
|   | 1 | 2 |
+---+---+---+
*/

emptyTicket.cells(in: .horizontal(.bottom))
// Returns [nil, 1, 2]

emptyTicket.cells(in: .diagonalDownward)
// returns [7, 5, 2]

emptyTicket.cells(in: .vertical(.left))
// returns [7, nil, nil]
```

I think this is a good stopping point. I've created a data type with enough convenience functions to intuitively and ergonimically access the ticket and updates it's values. In the next part, I'll build the scoring and probability system.