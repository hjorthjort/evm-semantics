EVM World State
===============

We need a way to specify the current world state. It will be a list of accounts
and a list of pending transactions. This can come in either the pretty K format,
or in the default EVM test-set format.

First, we build a JSON parser, then we provide some standard "parsers" which
will be used to convert the JSON formatted input into the prettier K format.

```k
requires "data.k"

module EVM-WORLD-STATE
    imports EVM-DASM

    configuration <worldState>
                    <control> $WORLDSTATE:K </control>
                    <accounts>
                      <account multiplicity="*">
                        <acctID> .AcctID </acctID>
                        <nonce> 0:Word </nonce>
                        <balance> 0:Word </balance>
                        <program> .Map </program>
                        <storage> .Map </storage>
                      </account>
                    </accounts>
                    <transactions>
                      <transaction multiplicity="*">
                        <type> .TXType </type>
                        <to> .AcctID </to>
                        <from> .AcctID </from>
                        <data> .WordStack </data>
                        <value> 0:Word </value>
                        <txGasPrice> 0:Word </txGasPrice>
                        <txGasLimit> 0:Word </txGasLimit>
                      </transaction>
                    </transactions>
                  </worldState>

    syntax KItem ::= Control
    syntax Control ::= "#response" Word
                     | Account | Accounts | Transaction | Transactions
                     | "accounts" ":" Accounts "transactions" ":" Transactions
 // --------------------------------------------------------------------------
    rule <control> accounts : ACCTS transactions : TXS => ACCTS ~> TXS ... </control>

    syntax AcctID ::= Word | ".AcctID"
 // ----------------------------------

    syntax TXType ::= "message" | "creation" | ".TXType"
 // ----------------------------------------------------
```

Here is the data of an account on the network. It has an id, a balance, a
program, and storage. Additionally, the translation from the JSON account format
to the K format is provided.

```k
    syntax Account ::= JSON
                     | "account" ":" "-" "id"      ":" AcctID
                                     "-" "nonce"   ":" Word
                                     "-" "balance" ":" Word
                                     "-" "program" ":" OpCodes
                                     "-" "storage" ":" WordStack
 // ------------------------------------------------------------
    rule <control> ACCTID : { "balance" : (BAL:String)
                            , "code"    : (CODE:String)
                            , "nonce"   : (NONCE:String)
                            , "storage" : STORAGE
                            }
                => account : - id      : #parseHexWord(ACCTID)
                             - nonce   : #parseHexWord(NONCE)
                             - balance : #parseHexWord(BAL)
                             - program : #dasmOpCodes(#parseWordStack(CODE))
                             - storage : #parseWordStack(STORAGE)
                ...
         </control>
```

Here is the data of a transaction on the network. It has fields for who it's
directed toward, the data, the value transfered, and the gas-price/gas-limit.
Similarly, a conversion from the JSON format to the pretty K format is provided.

```k
    syntax Transaction ::= JSON
                         | "transaction" ":" "-" "to"       ":" AcctID
                                             "-" "from"     ":" AcctID
                                             "-" "data"     ":" WordStack
                                             "-" "value"    ":" Word
                                             "-" "gasPrice" ":" Word
                                             "-" "gasLimit" ":" Word
                         | "transaction" ":" "-" "to"       ":" AcctID
                                             "-" "from"     ":" AcctID
                                             "-" "init"     ":" WordStack
                                             "-" "value"    ":" Word
                                             "-" "gasPrice" ":" Word
                                             "-" "gasLimit" ":" Word
 // ----------------------------------------------------------------------
    rule <control> "transaction" : { "data"      : (DATA:String)
                                   , "gasLimit"  : (LIMIT:String)
                                   , "gasPrice"  : (PRICE:String)
                                   , "nonce"     : (NONCE:String)
                                   , "secretKey" : (SECRETKEY:String)
                                   , "to"        : (ACCTTO:String)
                                   , "value"     : (VALUE:String)
                                   }
                => transaction : - to       : #parseHexWord(ACCTTO)
                                 - from     : .AcctID
                                 - data     : #parseWordStack(DATA)
                                 - value    : #parseHexWord(VALUE)
                                 - gasPrice : #parseHexWord(PRICE)
                                 - gasLimit : #parseHexWord(LIMIT)
                ...
         </control>
```

Finally, we have the syntax of an `EVMSimulation`, which consists of a list of
accounts followed by a list of transactions.

```k
    syntax Accounts ::= ".Accounts"
                      | Account Accounts
 // ------------------------------------
    rule <control> .Accounts => .                               ... </control>
    rule <control> ACCT:Account ACCTS:Accounts => ACCT ~> ACCTS ... </control>

    syntax Transactions ::= ".Transactions"
                          | Transaction Transactions
 // ------------------------------------------------
    rule <control> .Transactions => .                           ... </control>
    rule <control> TX:Transaction TXS:Transactions => TX ~> TXS ... </control>
endmodule
```

State Getters/Setters
---------------------

We'll make some getters/setters for the world-state. This can be considered the
API or "system calls" for the EVM world state.

```k
module EVM-WORLD-STATE-INTERFACE
    imports EVM-WORLD-STATE

    syntax Control ::= Account | Transaction
 // ----------------------------------------
    rule <control> account : - id      : ACCT
                             - nonce   : NONCE
                             - balance : BAL
                             - program : PROGRAM
                             - storage : STORAGE
                 => . ...
         </control>
         <accounts>
            ( .Bag
           => <account>
                <acctID> ACCT </acctID>
                <nonce> NONCE </nonce>
                <balance> BAL </balance>
                <program> #asMap(PROGRAM) </program>
                <storage> #asMap(STORAGE) </storage>
              </account>
            )
            ...
         </accounts>
      requires BAL >Int 0

    rule <control> transaction : - to       : ACCTTO
                                 - from     : ACCTFROM
                                 - data     : DATA
                                 - value    : VALUE
                                 - gasPrice : PRICE
                                 - gasLimit : LIMIT
                 => . ...
         </control>
         <transactions>
           ( .Bag
          => <transaction>
               <type> message </type>
               <to> ACCTTO </to>
               <from> ACCTFROM </from>
               <data> DATA </data>
               <value> VALUE </value>
               <txGasPrice> PRICE </txGasPrice>
               <txGasLimit> LIMIT </txGasLimit>
             </transaction>
           )
           ...
         </transactions>

    rule <control> transaction : - to       : ACCTTO
                                 - from     : ACCTFROM
                                 - init     : PROGRAM
                                 - value    : VALUE
                                 - gasPrice : PRICE
                                 - gasLimit : LIMIT
                 => . ...
         </control>
         <transactions>
           ( .Bag
          => <transaction>
               <type> creation </type>
               <to> ACCTTO </to>
               <from> ACCTFROM </from>
               <data> PROGRAM </data>
               <value> VALUE </value>
               <txGasPrice> PRICE </txGasPrice>
               <txGasLimit> LIMIT </txGasLimit>
             </transaction>
           )
           ...
         </transactions>

    syntax Control ::= "#transfer" AcctID AcctID Word
 // -------------------------------------------------
    rule <control> #transfer ACCTFROM ACCTTO BAL => . ... </control>
         <account>
           <acctID> ACCTFROM </acctID>
           <balance> BALFROM => BALFROM -Word BAL </balance>
           ...
         </account>
         <account>
           <acctID> ACCTTO </acctID>
           <balance> BALTO => BALTO +Word BAL </balance>
           ...
         </account>
      requires BALFROM >Int BAL

    rule <control> #transfer .AcctID ACCTTO BAL => . ... </control>
         <account>
           <acctID> ACCTTO </acctID>
           <balance> BALTO => BALTO +Word BAL </balance>
           ...
         </account>

    syntax Control ::= "#setAccountStorage" AcctID Word Word
                     | "#getAccountStorage" AcctID Word
 // ---------------------------------------------------
    rule <control> #setAccountStorage ACCT INDEX VALUE => . ... </control>
         <account>
           <acctID> ACCT </acctID>
           <storage> STORAGE => STORAGE[INDEX <- VALUE] </storage>
           ...
         </account>

    rule <control> #getAccountStorage ACCT INDEX => #response VALUE ... </control>
         <account>
           <acctID> ACCT </acctID>
           <storage> ... INDEX |-> VALUE ... </storage>
           ...
         </account>
endmodule
```

EVM Disassembler
================

The default EVM test-set format is JSON, where the data is hex-encoded. A
dissassembler is provided here for the basic data so that both the JSON and our
pretty format can be read in.

```k
module EVM-DASM
    imports EVM-OPCODE
    imports STRING

    syntax JSONList ::= List{JSON,","}
    syntax JSON     ::= String
                      | String ":" JSON
                      | "{" JSONList "}"
                      | "[" JSONList "]"
 // ------------------------------------

    syntax Word ::= #parseHexWord ( String ) [function]
 // ---------------------------------------------------
    rule #parseHexWord("")   => 0
    rule #parseHexWord("0x") => 0
    rule #parseHexWord(S)    => String2Base(replaceAll(S, "0x", ""), 16)
      requires (S =/=String "") andBool (S =/=String "0x")

    syntax WordStack ::= #parseHexWords  ( String ) [function]
                       | #parseWordStack ( String ) [function]
 // ----------------------------------------------------------
    rule #parseWordStack(S) => #parseHexWords(replaceAll(S, "0x", ""))
    rule #parseHexWords("") => .WordStack
    rule #parseHexWords(S)  => #parseHexWord(substrString(S, 0, 2)) : #parseHexWords(substrString(S, 2, lengthString(S)))
      requires lengthString(S) >=Int 2

    syntax OpCode ::= #dasmOpCode ( Word ) [function]
 // -------------------------------------------------
    rule #dasmOpCode(   0 ) => STOP
    rule #dasmOpCode(   1 ) => ADD
    rule #dasmOpCode(   2 ) => MUL
    rule #dasmOpCode(   3 ) => SUB
    rule #dasmOpCode(   4 ) => DIV
    rule #dasmOpCode(   5 ) => SDIV
    rule #dasmOpCode(   6 ) => MOD
    rule #dasmOpCode(   7 ) => SMOD
    rule #dasmOpCode(   8 ) => ADDMOD
    rule #dasmOpCode(   9 ) => MULMOD
    rule #dasmOpCode(  10 ) => EXP
    rule #dasmOpCode(  11 ) => SIGNEXTEND
    rule #dasmOpCode(  16 ) => LT
    rule #dasmOpCode(  17 ) => GT
    rule #dasmOpCode(  18 ) => SLT
    rule #dasmOpCode(  19 ) => SGT
    rule #dasmOpCode(  20 ) => EQ
    rule #dasmOpCode(  21 ) => ISZERO
    rule #dasmOpCode(  22 ) => AND
    rule #dasmOpCode(  23 ) => EVMOR
    rule #dasmOpCode(  24 ) => XOR
    rule #dasmOpCode(  25 ) => NOT
    rule #dasmOpCode(  26 ) => BYTE
    rule #dasmOpCode(  32 ) => SHA3
    rule #dasmOpCode(  48 ) => ADDRESS
    rule #dasmOpCode(  49 ) => BALANCE
    rule #dasmOpCode(  50 ) => ORIGIN
    rule #dasmOpCode(  51 ) => CALLER
    rule #dasmOpCode(  52 ) => CALLVALUE
    rule #dasmOpCode(  53 ) => CALLDATALOAD
    rule #dasmOpCode(  54 ) => CALLDATASIZE
    rule #dasmOpCode(  55 ) => CALLDATACOPY
    rule #dasmOpCode(  56 ) => CODESIZE
    rule #dasmOpCode(  57 ) => CODECOPY
    rule #dasmOpCode(  58 ) => GASPRICE
    rule #dasmOpCode(  59 ) => EXTCODESIZE
    rule #dasmOpCode(  60 ) => EXTCODECOPY
    rule #dasmOpCode(  64 ) => BLOCKHASH
    rule #dasmOpCode(  65 ) => COINBASE
    rule #dasmOpCode(  66 ) => TIMESTAMP
    rule #dasmOpCode(  67 ) => NUMBER
    rule #dasmOpCode(  68 ) => DIFFICULTY
    rule #dasmOpCode(  69 ) => GASLIMIT
    rule #dasmOpCode(  80 ) => POP
    rule #dasmOpCode(  81 ) => MLOAD
    rule #dasmOpCode(  82 ) => MSTORE
    rule #dasmOpCode(  83 ) => MSTORE8
    rule #dasmOpCode(  84 ) => SLOAD
    rule #dasmOpCode(  85 ) => SSTORE
    rule #dasmOpCode(  86 ) => JUMP
    rule #dasmOpCode(  87 ) => JUMPI
    rule #dasmOpCode(  88 ) => PC
    rule #dasmOpCode(  89 ) => MSIZE
    rule #dasmOpCode(  90 ) => GAS
    rule #dasmOpCode(  91 ) => JUMPDEST
    rule #dasmOpCode( 240 ) => CREATE
    rule #dasmOpCode( 241 ) => CALL
    rule #dasmOpCode( 242 ) => CALLCODE
    rule #dasmOpCode( 243 ) => RETURN
    rule #dasmOpCode( 244 ) => DELEGATECALL
    rule #dasmOpCode( 254 ) => INVALID
    rule #dasmOpCode( 255 ) => SELFDESTRUCT

    syntax OpCodes ::= #dasmPUSH ( Word , WordStack ) [function]
                     | #dasmLOG  ( Word , WordStack ) [function]
    syntax Word ::= #pushArg ( WordStack ) [function]
 // -------------------------------------------------
    rule #dasmPUSH( W , WS ) => PUSH(#pushArg(#take(W, WS))) ; #dasmOpCodes(#drop(W, WS))

    syntax OpCodes ::= #dasmOpCodes ( WordStack ) [function]
 // --------------------------------------------------------
    rule #dasmOpCodes( .WordStack ) => .OpCodes
    rule #dasmOpCodes( W : WS )     => #dasmOpCode(W)    ; #dasmOpCodes(WS) requires (W >=Word 0   ==K bool2Word(true)) andBool (W <=Word 95  ==K bool2Word(true))
    rule #dasmOpCodes( W : WS )     => #dasmOpCode(W)    ; #dasmOpCodes(WS) requires (W >=Word 240 ==K bool2Word(true)) andBool (W <=Word 255 ==K bool2Word(true))
    rule #dasmOpCodes( W : WS )     => DUP( W -Word 127) ; #dasmOpCodes(WS) requires (W >=Word 128 ==K bool2Word(true)) andBool (W <=Word 143 ==K bool2Word(true))
    rule #dasmOpCodes( W : WS )     => SWAP(W -Word 143) ; #dasmOpCodes(WS) requires (W >=Word 144 ==K bool2Word(true)) andBool (W <=Word 159 ==K bool2Word(true))
    rule #dasmOpCodes( W : WS )     => #dasmLOG( W -Word 160 , WS )         requires (W >=Word 160 ==K bool2Word(true)) andBool (W <=Word 164 ==K bool2Word(true))
    rule #dasmOpCodes( W : WS )     => #dasmPUSH( W -Word 95 , WS )         requires (W >=Word 96  ==K bool2Word(true)) andBool (W <=Word 127 ==K bool2Word(true))
endmodule
```