
# elm-export-persistent

[![Build Status](https://travis-ci.org/jb55/elm-export-persistent.svg)](https://travis-ci.org/jb55/elm-export-persistent)

[elm-export](https://hackage.haskell.org/package/elm-export) helpers for
Persistent Entities.

## Usage

```hs
newtype Ent (field :: Symbol) a = Ent (Entity a)
  deriving (Generic)

type EntId a = Ent "id" a
```

Ent is a newtype that wraps Persistent Entity's, allowing you to export them to
Elm types. Specifically, it adds a {To,From}JSON instance which adds an `id`
field, as well as an `ElmType` instance that adds an `id` field constructor.

## Example

Let's define a Persistent model:

```hs
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE StandaloneDeriving #-}

module Main where

import Data.Aeson
import Data.Text
import Database.Persist.TH
import Elm
import GHC.Generics

import Elm.Export.Persist
import Elm.Export.Persist.BackendKey ()

share [mkPersist sqlSettings, mkMigrate "migrateAccount"] [persistLowerCase|
Account
  email Text
  password Text
  deriving Show Generic
  UniqueEmail email
|]

instance ToJSON Account
instance FromJSON Account
instance ElmType Account

-- use GeneralizedNewtypeDeriving for ids
-- this picks a simpler int-encoding
deriving instance ElmType AccountId 
```

Now let's export it with an id field:

```hs
module Main where

import Db
import Elm
import Data.Proxy

mkSpecBody :: ElmType a => a -> [Text]
mkSpecBody a =
  [ toElmTypeSource    a
  , toElmDecoderSource a
  , toElmEncoderSource a
  ]
  
defImports :: [Text]
defImports =
  [ "import Json.Decode exposing (..)"
  , "import Json.Decode.Pipeline exposing (..)"
  , "import Json.Encode"
  , "import Http"
  , "import String"
  ]

accountSpec :: Spec
accountSpec = 
  Spec ["Generated", "Account"] $
       defImports ++ mkSpecBody (Proxy :: Proxy (EntId Account))

main :: IO ()
main = specsToDir [accountSpec] "some/where/output"
```

This generates:

```elm
module Generated.Account exposing (..)

import Json.Decode exposing (..)
import Json.Decode.Pipeline exposing (..)
import Json.Encode
import Http
import String

type alias Account =
    { accountEmail : String
    , accountPassword : String
    , id : Int
    }

decodeAccount : Decoder Account
decodeAccount =
    decode Account
        |> required "accountEmail" string
        |> required "accountPassword" string
        |> required "id" int

encodeAccount : Account -> Json.Encode.Value
encodeAccount x =
    Json.Encode.object
        [ ( "accountEmail", Json.Encode.string x.accountEmail )
        , ( "accountPassword", Json.Encode.string x.accountPassword )
        , ( "id", Json.Encode.int x.id )
        ]
```
