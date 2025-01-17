---
title: Turing Machine
version: 0.1.0
description: Turing Machine in sCrypt
---

In the example below, we implement a Turing machine for [checking balanced parentheses](https://math.stackexchange.com/questions/503853/how-do-you-argue-or-prove-that-a-certain-turing-machine-accepts-a-language). Its initial state is A and it contains one accepted state. The transition function says, for instance, if the machine is at state A and its head reads symbol “)”, it should write “X” in that cell and move left, transitioning to state B.

Structs can be defined like the following:
```javascript
import "serializer.scrypt";
import "arrayUtil.scrypt";


// Turing machine state
type State = bytes;
// alphabet symbol in each cell, 1 byte long each
type Symbol = bytes;

// contract state as a struct
struct StateStruct {
    int headPos;
    bytes tape;
    // current machine state
    State curState;
}

struct Input {
    State oldState;
    Symbol read;
}

struct Output {
    State newState;
    Symbol write;
    // move left or right
    bool moveLeft;
}

// transition function entry: input -> output
struct TransitionFuncEntry {
    Input input;
    Output output;
}

/*
 * A Turing Machine checking balanced parentheses
 */
contract TuringMachine {

    @state
    StateStruct states;
    // states
    static const State STATE_A = b'00'; // initial state
    static const State STATE_B = b'01';
    static const State STATE_C = b'02';
    static const State STATE_ACCEPT = b'03';

    // symbols
    static const Symbol BLANK = b'00';
    static const Symbol OPEN = b'01';
    static const Symbol CLOSE = b'02';
    static const Symbol X = b'03';

    static const bool LEFT = true;
    static const bool RIGHT = false;

    // number of rules in the transition function
    static const int N = 8;
    // transition function table
    static const TransitionFuncEntry[N] transitionFuncTable = [
        {{STATE_A, OPEN},   {STATE_A, OPEN, RIGHT}},
        {{STATE_A, X},      {STATE_A, X, RIGHT}},
        {{STATE_A, CLOSE},  {STATE_B, X, LEFT}},
        {{STATE_A, BLANK},  {STATE_C, BLANK, LEFT}},
        
        {{STATE_B, OPEN},   {STATE_A, X, RIGHT}},
        {{STATE_B, X},      {STATE_B, X, LEFT}},

        {{STATE_C, X},      {STATE_C, X, LEFT}},
        {{STATE_C, BLANK},  {STATE_ACCEPT, BLANK, RIGHT}}
    ];
    
    public function transit(SigHashPreimage txPreimage) {
        // read/deserialize contract state
        SigHashType sigHashType = SigHash.ANYONECANPAY | SigHash.SINGLE | SigHash.FORKID;
        require(Tx.checkPreimageSigHashType(txPreimage, sigHashType));
        // transition
        Symbol head = ArrayUtil.getElemAt(this.states.tape, this.states.headPos);
        // look up in transition table
        bool found = false;
        loop (N) : i {
            if (!found) {
                auto entry = transitionFuncTable[i];
                if (entry.input == { this.states.curState, head }) {
                    auto output = entry.output;
                    // update state
                    this.states.curState = output.newState;
                    // write tape head
                    this.states.tape = ArrayUtil.setElemAt(this.states.tape, this.states.headPos, output.write);
                    // move head
                    this.states.headPos += output.moveLeft ? -1 : 1;
                    // extend tape if out of bound
                    if (this.states.headPos < 0) {
                        // add 1 blank cell to the left
                        this.states.tape = BLANK + this.states.tape;
                        this.states.headPos = 0;
                    }
                    else if (this.states.headPos >= len(this.states.tape)) {
                        // add 1 blank cell to the right
                        this.states.tape = this.states.tape + BLANK;
                    }

                    if (this.states.curState == STATE_ACCEPT) {
                        // accept
                        exit(true);
                    }

                    found = true;
                }
            }
        }
        // reject if no transition rule found
        require(found);

        // otherwise machine goes to the next step
        bytes stateScript = this.getStateScript();
        bytes output = Utils.buildOutput(stateScript, SigHash.value(txPreimage));
        require(hash256(output) == SigHash.hashOutputs(txPreimage));

    }
}
```

