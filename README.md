streaming-bytestring
====================
[![Hackage](https://img.shields.io/hackage/v/streaming-bytestring.svg)](https://hackage.haskell.org/package/streaming-bytestring) [![Build Status](https://travis-ci.org/haskell-streaming/streaming-bytestring.svg?branch=master)](https://travis-ci.org/haskell-streaming/streaming-bytestring)

This package depends on the [`streaming` library](https://github.com/haskell-streaming/streaming)


              copy 200M file    divide it on lines, 
                                adding '!' to each 
                                
    lazy      0m0.813s          0m8.597s
    streaming 0m0.783s          0m9.664s
    pipes     0m0.771s          0m49.176s
    conduit	  0m1.068s          2m25.437s

This library is modeled as far as possible on the internal structure of
`Data.ByteString.Lazy`. There are two changes: a chunk may be delayed
by a monadic step, and the sucession of steps has a 'return' value:

```haskell
data ByteString m r
  = Empty r
  | Chunk {-# UNPACK #-} !S.ByteString (ByteString m r)
  | Go (m (ByteString m r ))
```
unlike 
```haskell
data ByteString
  = Empty 
  | Chunk {-# UNPACK #-} !S.ByteString ByteString
```  
That's it. 

-----

Another module is planned that would correspond more closely to 
`Pipes.Bytestring` than to `Data.ByteString.Lazy`.   
`Producer ByteString m r` as it is treated in `pipes-bytestring` as
the `ByteString m r` type is here. The result is much faster, at least 
with preliminary tests. The modules integrating `attoparsec` and `aeson` 
are simple replicas of k0001's `pipes-attoparsec` and `pipes-aeson`. 
Also included is a replica of `pipes-http`.

It is possible that `streaming-bytestring` is conceptually clearer than 
`pipes-bytestring` as well - and clearer than the approach taken by 
`conduit` and `io-streams`.  All of these are forced to integrate the 
conception of *an amorphous succession of bytes that may be chunked anywhere* - 
the direct result of, say, `fromHandle`, `sourceFile` and
the like - and a succession of 'semantically' distinct bytestrings 
of interest under a single concept. 

----

Strange as it may seem, it is arguable that the general `Producer`, 
`Source`, and `InputStream` concepts from these libraries ought not 
to hold `ByteString`s *except* as conceptually separate units, e.g. 
the lines of a document taken as strict bytestrings, where that is 
legitimate. An `InputStream ByteString` is like an `InputStream Int`; 
a `Conduit.Source m ByteString` has the same type as a `Source m Int`;
a `Pipes.Producer ByteString m r` has the same type as a `Producer Int m r`.
These types are suited to the general stream transformations these 
libraries make possible. 

We can see the strangeness in the `io-streams` `lines` 

    lines :: InputStream ByteString -> IO (InputStream ByteString)

and the `conduit` `linesUnboundedAscii`

    linesUnboundedAscii :: (Monad m) => Conduit ByteString m ByteString
    
(specializing slightly). In either case, what enters on the left will
be a succession of anyhow-chunked bytes; what exits on the right will 
be a succession of significant individual things of type `ByteString`.  

What we find in `IOStreams.lines` and
`linesUnlimitedAscii` are comparable to what we would have if `bytestring`
defined 

    lines :: L.ByteString -> [S.ByteString]
   
or more absurdly

    lines :: L.ByteString -> L.ByteString 

and exposed methods for inspecting the hitherto secret chunks contained
in lazy bytestrings. 

The model employed by the present package is a little different.  First, 
the primitive `lines` concept is just

    lines :: ByteString m r -> Stream (ByteString m) m r

as in `pipes-bytestring`; this corresponds precisely to 

    lines :: ByteString -> [ByteString]

as it appears in `Data.ByteString.Lazy` -- the elements of the list (stream) are 
themselves lazy bytestrings. 

But `pipes-bytestring` attempts to *mean* by `Producer ByteString m r` 
what we express by `ByteString m r` - the undifferentiated byte stream.
But (we are provisionally suggesting) that isn't what `Producer ByteString m r` 
means, and this is part of the reason why `pipes-bytestring` is difficult 
for people to grasp. The user frequently proposes to inspect and work 
with individual lines with Pipes themselves and thus needs

    produceLines :: Producer ByteString m r -> Producer ByteString m r
    produceLines = folds B.concat B.empty id . view Pipes.ByteString.lines
    
Here we would instead write a 

    produceLines :: ByteString m r -> Stream (Of ByteString) m r

which is transparently related to the type of lines itself

    lines :: ByteString m r -> Stream (ByteString m) m r

The distinctive type of `produceLines` clearly express the transition 
from the world of amorphously chunked bytestreams to the world of 
significant individual values, in this case individual strict bytestrings.  



