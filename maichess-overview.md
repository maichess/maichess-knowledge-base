# maichess

*maichess (my/ai-chess)* is an academic project with the goal of entirely vibe coding a complete chess application infrastructure. 

The focus lies on architectural principles for serving a realtime web-app to a large amount of users reliably and fast — and less on macro-code-quality or going new ways. 
The domain of a chess-application was explicitly chosen because it is a saturated one with lots of training data.

## From native monoliths...

Initially, a monolithic native app was built using scala to get a feel for the domain, featuring:

- A fully interactable console-based user interface, synchronous with a scalaFX graphical interface.
- Multiple chess engine implementations of variying degrees of difficulty as computer-controlled opponents.
- FEN, PGN and SAN import/export as well as undo/redo functionality to analyze games.
- A REST API interface for interacting with the game through HTTP requests.

## ... to microservices in the web.

The ultimate goal however is a microservice-based web-app that can scale to potentially millions of users. The proposed microservice structure is outlined further in [maichess-structure](maichess-structure.md).

When users visit the app for the first time they will be prompted to create an account or log in via typical OAuth providers like Google or GitHub. 
After account creation they will be able to enter match-making to find other players to play against, play against one of the supported chess-engines or even just watch two chess engines play against each other.