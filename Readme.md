
# Quoridor Client

## 1. Rules Introduction

**Data:**  
The horizontal direction is the $x$ axis increasing to the right, and the vertical direction is the $y$ axis increasing upwards. The board is $9 \times 9$, with piece coordinates ranging from integers in $[1,9]$, making a total of 81 piece positions, with coordinates $(1...9, 1...9)$. Walls are represented by start and end coordinates, with coordinate ranges as integers in $[0,9]$. Walls can be placed within all trenches except the outermost boundaries and must be either horizontal or vertical, with a length of 2 squares. For example, the wall $((4,1), (6,1))$ represents a wall above the piece coordinates $(5,1)$ and $(6,1)$ and below $(5,2)$ and $(6,2)$. In this game, walls cannot overlap (e.g., after placing $((4,1), (6,1))$, $((3,1), (5,1))$ and $((5,1), (7,1))$ cannot be placed), but they can intersect (e.g., after placing $((4,1), (6,1))$, $((5,0), (5,2))$ can be placed).

**Rules:**
1. A player wins by reaching any position on the opposite side of the board from their starting position (if starting at $(5,1)$, reaching $(1...9, 9)$ wins; if starting at $(5,9)$, reaching $(1...9, 1)$ wins).
2. Three violations result in an automatic loss.
3. If a player neither moves their piece nor places a wall, it is considered a no-action move, which counts as a violation.
4. Placing a wall that conflicts with existing walls (overlapping, blocking the opponent completely, or having incorrect coordinates) counts as a violation.
5. Moving a piece and placing a wall in the same turn is considered a move action.
6. If the move location is unreachable, it counts as a violation.
7. If the step calculation time (executing `nextStep()` plus communication time) exceeds the timeout limit (tentatively set at ≥1 second), it counts as a violation. (The debugging server does not check for timeouts.)
8. Accumulating three violations results in an automatic loss.
9. To prevent indefinite back-and-forth moves leading to a stalemate, the game limits each player to a maximum of 100 moves per match. If the move limit is exceeded, the player with the shortest total time wins.
10. During the competition, players exceeding the time limit are immediately disqualified.

## 2. Game Server Environment Setup

### JRE Installation:

Download 'jre-8u231-windows-x64.exe' from the provided file, follow the installation instructions without modifying any settings (everything should be installed with default settings). After installation, open the Command Prompt (CMD) and type `java` to verify the installation. If no command-not-found error appears, the installation is successful. This step is then complete.

`For server documentation, refer to the README.md file in the server directory.`

## 3. Notes

### 3.0. Initial Test (Windows + Visual Studio) Graphical Guidance Reference `Quoridor Initial Test.pptx`

1. After obtaining the client code, directly double-click `Quoridor\bin\QuoridorUI.exe` on Windows to check if it correctly replays the test log file.
2. Without modifying any code:
    1. Install and configure JDK1.8. In the directory containing `server.jar`, execute `java -jar server.jar -p 19330 -n 2` from the command line to run the server executable.
    2. Double-click to run `Quoridor\bin\BaselineClient.exe`.
    3. Open `Quoridor\Quoridor.sln` in Visual Studio 2019, compile, and run the solution to familiarize yourself with the development workflow.

### 3.1. Compilation Notes

If developing only with Visual Studio, this section can be skipped.

1. If Visual Studio compilation triggers security warnings for `printf_s`, add the macro definition `#define _CRT_SECURE_NO_WARNINGS`.
2. In Windows using the MinGW environment, navigate to the `Quoridor\` directory and run `make` to generate the executable in `Quoridor\bin`.
3. For Linux, replace `Quoridor/makefile_linux` with `Quoridor/makefile` and then run `make`.
4. Due to system differences, `make clean` is not implemented in the makefile.

### 3.2. Auto Replay Configuration

The auto replay mechanism is implemented via terminal commands. Ensure `Quoridor\bin\QuoridorUI.exe` and the UI configuration files (`Quoridor\bin\uiconfig.json` and `Quoridor\bin\panel`) are in a directory where `QuoridorClient` can find them directly (for debugging in Visual Studio, place them in `Quoridor\QuoridorClient\`).

### 3.3. 

`QuoridorUI.exe` has two optional input parameters. Modifying its configuration file for log file reading is not recommended. See the `getConfig()` function in `Quoridor\QuoridorUI\QuoridorUI.cpp` for details.

```
QuoridorUI.exe                  # Reads the default uiconfig.json configuration file and replays the log file specified in it
QuoridorUI.exe log_xx_xx.csv    # Replays the log_xx_xx.csv log file
QuoridorUI.exe uiconfig.json    # Reads the uiconfig.json configuration file
QuoridorUI.exe log_xx_xx.csv uiconfig.json   # Reads uiconfig.json and replays log_xx_xx.csv
QuoridorUI.exe uiconfig.json log_xx_xx.csv   # Same as above
```

### 3.4. 

`Quoridor\QuoridorUI\uiconfig.json` must be used together with `Quoridor\QuoridorUI\panel`.


## 4. QuoridorUtils Basic Game Data Structures

**Please carefully read the source file `Quoridor\QuoridorUtils\QuoridorUtils.h`**

### 4.0. Data Structure Overview

`QuoridorUtils.h` contains a total of one enumeration class `GameStatus` and four structures: `Location`, `BlockBar`, `ChessboardChange`, and `Step`.

Note: All enumeration classes and structures are within the `namespace QuoridorUtils`. If using these structures in other namespaces, use identifiers like `QuoridorUtils::Location` or add `using namespace QuoridorUtils;`.

### 4.1. Constants and GameStatus Enumeration Class

Some game status flags indicate "no player handling required," meaning these statuses are processed within `QuoridorUtils::Configuration`. For example, when a player wins, `QuoridorUtils::Configuration` will automatically handle the victory message, wait for a restart, and call the player's `restart()` function to reset the player state.

```c++
extern const int SIZE;        // Board size: 9
extern const int BLOCK_SUM;   // Total number of walls available to a player: 10
extern const int BLOCK_LEN;   // Wall length: 2

enum class GameStatus {
    Ok = 0,                   // Normal
    Win = 1,                  // Victory, no player handling required
    Lost = 2,                 // Defeat, no player handling required
    Timeout = 3,              // Timeout
    EnemyClosed = 4,          // Opponent error, no player handling required
    RulesBreaker = 5,         // Multiple rule violations result in loss, no player handling required
    InsufficientBlock = 6,    // No walls left
    InvalidStep = 7,          // Invalid move or incorrect wall placement
    None = 2000,              // Undefined state, do not use
};
```

### 4.2. Location Coordinates

This structure provides the fundamental coordinate structure for the program. The default initialization coordinates are $(-1, -1)$, which indicates an empty data state in the game. The valid board coordinate range for $x$ and $y$ is $[1, SIZE]$ (inclusive). See the PPT for coordinate definitions. The `==` operator is overloaded for equality comparison of two coordinates (`a==b`). The `!=` operator is not provided; use `!(a==b)` instead. The constants `PLAYER0_LOC` and `PLAYER1_LOC` represent the starting positions of the two players, and players must determine their own initial position.

```c++
struct Location {
    int x;
    int y;
    Location(const int x = -1, const int y = -1) {
        this->x = x;
        this->y = y;
    };
    bool operator==(const Location& rLoc) const {
        return this->x == rLoc.x && this->y == rLoc.y;
    }
    int distance(const Location& rLoc) const {  // Calculate Manhattan distance
        return (abs(this->x - rLoc.x) + abs(this->y - rLoc.y));
    }
};

extern const Location PLAYER0_LOC;  // Player 0's starting position (5, 1)
extern const Location PLAYER1_LOC;  // Player 1's starting position (5, SIZE)

// Example code
Location myLoc;          // My position (-1, -1)
Location enemyLoc(5,1);  // Enemy position (5, 1)
myLoc = PLAYER1_LOC;     // My position (5, 9)
if (!(myLoc == enemyLoc)) {  // Check if two coordinates are different
    std::cout << "City Block distance: " << myLoc.distance(enemyLoc) << std::endl;
} else {
    std::cout << "equivalence. \n";
}
```

### 4.3. BlockBar (Wall)

Walls are represented by a start and end coordinate pair. See the PPT for examples. The valid coordinate range for walls is $[0, SIZE]$, and their length is always `BLOCK_LEN`. Any invalid wall can be detected using `blockBar.isNan()` which returns `true`. Use `isH()` to check if a wall is horizontal and `isV()` to check if it is vertical. The `normalization()` function standardizes the wall data, ensuring two walls are correctly compared.

```c++
struct BlockBar {
    Location start;  // Wall start coordinate
    Location stop;   // Wall end coordinate
    BlockBar(const int startX = -1, const int startY = -1, 
             const int stopX = -1, const int stopY = -1) {
        this->start.x = startX;
        this->start.y = startY;
        this->stop.x = stopX;
        this->stop.y = stopY;
    };
    BlockBar(const Location start, const Location stop = Location(-1, -1)) {
        this->start = start;
        this->stop = stop;
    };
    bool operator==(const BlockBar& rLoc) const {
        return this->start == rLoc.start && this->stop == rLoc.stop;
    }
    bool isNan() const {  // Checks if a wall is valid, returns true if invalid
        return this->start.x < 0 || this->start.x > SIZE ||
               this->start.y < 0 || this->start.y > SIZE || 
               this->stop.x < 0 || this->stop.x > SIZE ||
               this->stop.y < 0 || this->stop.y > SIZE || 
               !(isH() || isV()) || 
               (start.distance(stop) != BLOCK_LEN);
    };
    bool isH() const { // Check if wall is horizontal
        return this->start.y == this->stop.y && this->start.x != this->stop.x;
    };
    bool isV() const { // Check if wall is vertical
        return this->start.x == this->stop.x && this->start.y != this->stop.y;
    };
    void normalization() {  // Standardize to ensure start < stop, may swap start and stop
        if (this->start.x >= this->stop.x && this->start.y >= this->stop.y) {
            const Location tmp = this->start;
            this->start = this->stop;
            this->stop = tmp;
        }
    }
};
```

```c++
// Example code
BlockBar aBlock(4, 1, 6, 1);
BlockBar bBlock(6, 1, 4, 1);
if ((!aBlock.isNan()) && (!bBlock.isNan())) {
    aBlock.normalization();  // (4, 1, 6, 1)
    bBlock.normalization();  // (4, 1, 6, 1)
    if (aBlock == bBlock) {
        std::cout << "equivalence!" << std::endl;
    }
} else {
    std::cout << "Illegal BlockBar!" << std::endl;
}
```

### 4.4. ChessboardChange Board Changes

The ChessboardChange structure provides data on board changes to the player's `Step nextStep(ChessboardChange&)` function.

Specific operations involve the `Location` and `BlockBar` data structures. No example code is provided here.

```c++
struct ChessboardChange {
    GameStatus status = GameStatus::Ok;  // Status of my last move
    Location myLoc;                      // My current piece position
    Location enemyLoc;                   // Opponent's current piece position
    BlockBar newEnemyBlockBar;  // If the opponent placed a wall, this stores its start and end coordinates.
                                // If no wall was placed, defaults to (-1, -1, -1, -1) 
    bool isFinish() const {     // Check if the game has ended
        return isWin() || isLost();
    }

    bool isWin() const {        // Check if the player has won
        switch (this->status) {
        case GameStatus::EnemyClosed:
        case GameStatus::Win:
            return true;
        default:
            return false;
        }
    }

    bool isLost() const {       // Check if the player has lost
        switch (this->status) {
        case GameStatus::Lost:
        case GameStatus::RulesBreaker:
            return true;
        default:
            return false;
        }
    }
};
```

### 4.5. Step Player Moves

The `Step nextStep(ChessboardChange&)` function returns this data structure. The game allows only one type of action per turn (either move or place a wall). If both `myNewLoc` and `myNewBlockBar` are set to -1, it indicates no action was taken, and the server considers it a violation. If neither are -1, it means the player moved their piece.

Operations involve `Location` and `BlockBar`. No example code is provided here.

```c++
struct Step {
    Location myNewLoc;         // Target location of the player's move; defaults to (-1, -1) if not moving
    BlockBar myNewBlockBar;    // If the player places a wall, this stores the wall's start and end coordinates.
                               // Defaults to (-1, -1, -1, -1) if no wall is placed
    Step(const Location loc = Location(), const BlockBar block = BlockBar()) {
        this->myNewLoc = loc;
        this->myNewBlockBar = block;
    }
    Step(const ChessboardChange chessboard) {
        this->myNewLoc = chessboard.enemyLoc;
        this->myNewBlockBar = chessboard.newEnemyBlockBar;
    }
    bool isMove() const {      // If myNewLoc is within range, it is a move action
        return this->myNewLoc.x >= 1 && this->myNewLoc.x <= SIZE &&
               this->myNewLoc.y >= 1 && this->myNewLoc.y <= SIZE;
    }
    bool isNan() const {       // If both myNewLoc and myNewBlockBar are empty, no action was taken
        return !this->isMove() && this->myNewBlockBar.isNan();
    }
};
```

### 4.6. Others

The above enumeration class and four structures support standard stream output operations, allowing direct output with statements like `std::cout << loc`.

Hash functions are provided for `Location` and `BlockBarHash`, making them usable as keys in STL unordered containers.

```c++
extern std::ostream& operator<<(std::ostream& os, const QuoridorUtils::GameStatus status);
extern std::ostream& operator<<(std::ostream& os, const QuoridorUtils::Location loc);
extern std::ostream& operator<<(std::ostream& os, const QuoridorUtils::BlockBar block);
extern std::ostream& operator<<(std::ostream& os, const QuoridorUtils::ChessboardChange change);
extern std::ostream& operator<<(std::ostream& os, const QuoridorUtils::Step step);

struct LocationHash {  
    size_t operator()(const Location& loc) const {
        return std::hash<int>{}(loc.x) + std::hash<int>{}(loc.y);
    }
};
struct BlockBarHash {
    size_t operator()(const BlockBar& block) const {
        return std::hash<int>{}(block.start.x) + std::hash<int>{}(block.start.y) + 
            std::hash<int>{}(block.stop.x) + std::hash<int>{}(block.stop.y);
    }
};

// If using Location or BlockBar as keys in unordered_map/unordered_set, use these hashes:
std::unordered_map<QuoridorUtils::Location, valClass, QuoridorUtils::LocationHash> map_obj;
std::unordered_set<QuoridorUtils::BlockBar, QuoridorUtils::BlockBarHash> set_obj;
```

## 5. QuoridorClient Game Client

**Source files: Quoridor\QuoridorClient\MyPlayer.cpp(.h)**

The main algorithm functions are in `Quoridor\QuoridorClient\MyPlayer.cpp`.

### 5.1. Overall Client Framework

The main function of the client is implemented in `Client.cpp`. The implementation process includes: reading configuration files, connecting to the network, creating a player, starting the game, and replaying the game.

### 5.2. Programming Implementation

#### Object-Oriented Player Implementation

In the `MyPlayer.h` file, in addition to the three required functions (`MyPlayer(string)` constructor, `nextStep(ChessboardChange)` decision function, and `restart()` board state reset function), an `algorithmWalk(Location, Location)` function is also implemented. The `targetY` attribute stores the player's destination, and the `blocks` attribute stores all the walls. The walls in `blocks` are used to determine which directions are blocked at the player's current position.

### 5.3. Program Debugging

Since this program is relatively large, requiring a server and two players to participate, please ensure that you have completed **3.0. Initial Test** before debugging.

## 6. QuoridorUI Game Replay Program

The client UI is implemented using a console interface. Two abstract classes, `DataLoader` and `GameView`, are defined as interfaces on top of the game's underlying data structures in `QuoridorUtils.cpp/h`. `DataLoader` is an abstract class for loading data. In this project, its subclass `LogLoader` is implemented to load log files in CSV format. `GameView` is used to display the game board, and its subclass `ConsoleView` displays the board using character output.

Based on this, the `Relayer` class is defined as a game replay controller that connects the two interfaces, receives user input, controls the `DataLoader` object for input, and controls the `GameView` object for output.

### 6.1. DataLoader Virtual Data Loading Class

The code for this virtual class is as follows. It consists of two steps: 1. Read player names; 2. Read data line by line. If data is read successfully, it returns `true`; otherwise, it returns `false`.

```c++
#include <string>
#include "../QuoridorUtils/QuoridorUtils.h"

namespace QuoridorUtils {
class DataLoader {
public:
    virtual ~DataLoader() = default;
    // player0Name is the player starting at QuoridorUtils::PLAYER0_LOC;
    // player1Name is the player starting at QuoridorUtils::PLAYER1_LOC;
    virtual void getPlayerName(std::string& player0Name, std::string& player1Name) = 0;
    virtual bool getNextStep(std::string& playerName, QuoridorUtils::Step& nextStep, QuoridorUtils::GameStatus& status) = 0;
    virtual int getRemainingStep() = 0;
};
}
```

### 6.2. GameView Virtual Interface Display Class

Unlike `DataLoader`, `GameView` follows these steps: 1. Input player names; 2. Insert game steps. If the player name is incorrect, it returns `false`; 3. Display the previous and next steps. If displayed successfully, it returns `true`.

```c++
#include <string>
#include "../QuoridorUtils/QuoridorUtils.h"

namespace QuoridorUtils {
class GameView {
public:
    virtual ~GameView() = default;
    // Player registration, only two players can participate in the game.
    //  firstPlayer is the player starting at QuoridorUtils::PLAYER0_LOC;
    // secondPlayer is the player starting at QuoridorUtils::PLAYER1_LOC;
    virtual void playerRegister(const std::string& firstPlayer, const std::string& secondPlayer) = 0;
    // Insert the next move into the board. If playerName is not registered, the move is not added.
    virtual bool putStep(const std::string& playerName, const Step& nextStep, const GameStatus& status) = 0;
    virtual bool showNextStep() = 0;
    virtual bool showPrevStep() = 0;
};
}


