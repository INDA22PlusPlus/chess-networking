# Spec
This protocol specifies the interaction between two computers running two implementations of chess.

The protobuf file networking.proto contains the protobuf code. Either use the build crate ``prost_build`` to generate the rust code you need, or use the provided networking.rs file.

# Initial connection
The server exposes port 1337 for incoming connections. 
```proto
message C2SConnectRequest {
	optional uint64 game_id = 0;
	bool spectate = 1;
}
```
When a ``C2SConnectRequest`` (C2S: Client to server) packet is received, the server is to handle the start of a new game. It is up to the implementation to determine whether or not to accept spectators, or whether to allow concurrent connections. ``game_id`` is to be set if the client wishes to spectate or resume a game with a given ``game_id``.

```proto
message S2CConnectAck {
	bool success = 0;
	optional uint64 game_id = 1; 
	optional BoardState starting_position = 2;
	optional bool client_is_white = 3;
}


message BoardState {
	string fen_string = 0;
}
```

The server then responds with a ``S2CConnectAck`` (S2C: Server to client) packet. If the server allows the connection to be established, the ``success`` field will be ``true``. 
The ``game_id`` will be set to either a) the same ``game_id`` received in the ``C2SConnectRequest`` packet for either resuming a game or allowing a spectator or b) a value generated by the server if a new game is to be generated. 
The ``starting_position`` field contains the ``BoardState`` to start from, simply containing a FEN string representation of the starting position. This field can be used to allow players to resume from a previous position. 
**NOTE:** the client should be allowed to start as white if a new game is created, control this using ``client_is_white``

# Moves

```proto
enum Piece {
	Pawn = 0;
	Knight = 1;
	Bishop = 2;
	Rook = 3;
	Queen = 4;
	King = 5;
}

message Move {
	uint32 from_square = 0;
	uint32 to_square = 1;
	optional Piece promotion = 2;
}

```
When a player (either server or client) wishes to make a move, a ``Move`` packet is sent. The ``Move`` packet contains two unsigned integers ``from_square`` and ``to_square`` ranging from ``0-63``, with ``0`` being the a1 square and ``63`` being the h8 square. If the player wishes to promote, the ``Promotion`` field is set to the ``Piece`` (an enum) it wishes to promote to. 

```proto
message S2CMoveAck {
	bool legal = 0;
	BoardState board_result = 1; 
}
```

After receiving a move, the server is to send a ``S2CMoveAck`` packet. If the move was considered legal, the ``legal`` field should be set to ``true``, ``false`` if illegal. ``board_result`` should contain the board representation as the board looks for the server after the move has been played. The client **must** override their own position to this value if it differs from their own game state. 

**Authoritative server**: if the server finds the move to be illegal, it will ignore the move; set ``legal : false`` and send an unchanged ``board_result``, so that the client has to redo their move. On the other hand, if the client finds the servers move illegal, it will simply have to comply and perform the move anyways.


