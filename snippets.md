## Correct Integer Arithmetic

### Wrapping in Newtype

```haskell
precPrice :: Int
precPrice = 4

showUnit :: Integral a => a -> Int -> String
showUnit u p = show (fromIntegral u / (10^p) :: Double)

generateUnit :: Integral a => Double -> Int -> a
generateUnit u p = round $ u * (10^p)

newtype Price = Price
    { unPrice :: Word32
    } deriving (Eq, Ord, Num, Generic, FromJSON, ToJSON, Serialize)

price :: Double -> Price
price p = Price $ generateUnit p precPrice

instance Show Price where
    show (Price p) = showUnit p precPrice
```

### Safe Multiplication

```haskell
priceAmount :: Price -> Amount -> Volume
priceAmount (Price p) (Amount a) = Volume (p * a)
```

## Engine Monad

```haskell
newtype UpdateEngineT m a = UpdateEngineT
    { runUpdateEngineT :: StateT Market (EitherT String m) a
    } deriving ( Functor, Applicative, Monad
               , MonadIO, MonadState Market, MonadError String)

instance MonadTrans UpdateEngineT where
    lift = UpdateEngineT . lift . lift

newtype QueryEngineT m a = QueryEngineT
    { runQueryEngineT :: ReaderT Market (EitherT String m) a
    } deriving ( Functor, Applicative, Monad
               , MonadIO, MonadReader Market, MonadError String)

instance MonadTrans QueryEngineT where
    lift = QueryEngineT . lift . lift

type UpdateEngine = UpdateEngineT Identity
type QueryEngine = QueryEngineT Identity

runUpdateEngine :: Monad m => UpdateEngineT m a -> Market -> m (Either String (a, Market))
runUpdateEngine m s = runEitherT . runStateT (runUpdateEngineT m) $ s

runQueryEngine :: Monad m => QueryEngineT m a -> Market -> m (Either String a)
runQueryEngine m r = runEitherT . runReaderT (runQueryEngineT m) $ r

liftQuery :: Monad m => QueryEngineT m a -> UpdateEngineT m a
liftQuery m = do
    st <- get
    res <- lift $ runQueryEngine m st
    case res of
        Left s -> throwError s
        Right a -> return a
```

### Running the Engine

```haskell
processMessage :: Message -> Market -> (Event, Market)
processMessage (Message _ !message) !market =
  case message of
    Cmd c -> case update c market of
              Left s  -> (EvtError s, market)
              Right x -> x
    Qry q -> case query q market of
              Left s  -> (EvtError s, market)
              Right x -> (x, market)
    Evt _ -> (EvtEmpty, market)
    Halt  -> (EvtHalt, market)
  where
    update cmd = runIdentity . runUpdateEngine (runCommand cmd)
    query  q   = runIdentity . runQueryEngine (runQuery q)
```

## Commands and Interpreter

```haskell
data Command
  = CmdDeposit !User !Balance
  | CmdPlaceOrder !User !Side !Price !Amount
  | CmdCancelOrder !User !Id
  deriving (Show, Generic)

instance Serialize Command where
  put c = case c of
    CmdDeposit     u b     -> putWord8 0 >> put u >> put b
    CmdPlaceOrder  u s p a -> putWord8 1 >> put u >> put s >> put p >> put a
    CmdCancelOrder u i     -> putWord8 2 >> put u >> put i
  get = do
    tag <- getWord8
    case tag of
      0 -> CmdDeposit     <$> get <*> get
      1 -> CmdPlaceOrder  <$> get <*> get <*> get <*> get
      2 -> CmdCancelOrder <$> get <*> get
```

```haskell
runCommand :: Command -> UpdateEngine Event
runCommand c = case c of
    CmdDeposit u b -> deposit u b
    CmdPlaceOrder u s p a -> placeOrder u s p a
    CmdCancelOrder u i -> cancelOrder u i
```
