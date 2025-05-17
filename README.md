# factorio-calculator-haskell
Library for calculating ingredients and factories from required items on factorio.

WIP: Prototype sources are now on creating.

====== Descriptions for Prototype ======

こちらは、ライブラリ利用者が想像できるような「呼び出し例」をいくつか挙げたものです。実際の関数名やモジュール構成はお好みで変えていただければと思います。

---

## 1. シンプルな純関数スタイル

```haskell
{-# LANGUAGE OverloadedStrings #-}
import Factorio.RecipeCalc

main :: IO ()
main = do
  -- 「電子回路」を毎秒 2 個ほしい、という要件を記述
  let requirement = Requirement
        { reqRecipe = "electronic-circuit"
        , reqRate   = 2  -- pieces per second
        }
      -- 材料投入速度と機械台数を計算
      plan = calculatePlan defaultConfig requirement

  -- 出力例:
  -- MaterialRequirement { ironPlate = 4.0, copperCable = 8.0, ... }
  -- MachineCount        { machineType = "assembling-machine-2", count = 0.5 }
  print plan
```

* `calculatePlan` は純関数で、与えられた要件 (`Requirement`) と設定 (`Config`) から必要な材料投入速度や機械数をまとめた `Plan` を返します。

---

## 2. EDSL（組み込みDSL）風

```haskell
{-# LANGUAGE RebindableSyntax #-}
import Factorio.RecipeCalc.DSL

main :: IO ()
main = runCalc $ do
  -- DSL で「電子回路を毎秒 2 個」要求
  want "electronic-circuit" @ 2 / Sec

  -- レシピチェーン全体を解決して出力
  outputPlan
```

### DSL のイメージ

* `want :: RecipeName -> RateExpr -> CalcM ()`
* `(@)  :: RecipeName -> Double -> RateExpr`
* `(/)  :: RateExpr -> TimeUnit -> Rate`

```haskell
-- 内部的にはこんな感じ
data RateExpr = RateExpr RecipeName Double
data TimeUnit  = Sec | Min
type Rate      = Double  -- pieces per second

want "electronic-circuit" @ 2 / Sec
```

---

## 3. 複数レシピのバッチ処理

```haskell
import Factorio.RecipeCalc
import Data.Map (Map)

main :: IO ()
main = do
  -- 複数レシピの同時要求
  let requirements = Map.fromList
        [ ("electronic-circuit", 2)
        , ("iron-gear-wheel"   , 5)
        ]
      plan = batchPlan defaultConfig requirements

  -- 結果をテーブル表示
  mapM_ print (planMachineCounts plan)
  mapM_ print (planMaterialReqs    plan)
```

* `batchPlan :: Config -> Map RecipeName Rate -> PlanBatch`
* `planMachineCounts :: PlanBatch -> [(MachineType, Double)]`
* `planMaterialReqs    :: PlanBatch -> [(Item, Double)]`

---

## 4. コマンドラインインターフェース経由

```haskell
-- exe/Main.hs
{-# LANGUAGE DeriveGeneric #-}
import Options.Applicative
import Factorio.RecipeCalc

data Args = Args
  { recipe :: String
  , rate   :: Double
  }

parser :: Parser Args
parser = Args
  <$> strOption
        ( long "recipe"
       <> metavar "RECIPE"
       <> help "Recipe name, e.g. electronic-circuit" )
  <*> option auto
        ( long "rate"
       <> metavar "RATE"
       <> help "Desired output rate (per second)" )

main :: IO ()
main = do
  Args r q <- execParser (info parser fullDesc)
  let plan = calculatePlan defaultConfig (Requirement r q)
  putStrLn $ renderPlanText plan
```

```bash
$ myfactorio --recipe electronic-circuit --rate 2
# 出力例
-- Machines:
assembling-machine-2 x0.50
-- Materials / sec:
iron-plate      4.00
copper-cable    8.00
electronic-circuit 2.00
```

---

## 5. カスタム設定を差し替えて使う

```haskell
import Factorio.RecipeCalc

-- 独自の機械速度やモジュール効果を反映した設定
myConfig :: Config
myConfig = defaultConfig
  { machineSpeed = Map.fromList
      [ ("assembling-machine-1", 0.5)
      , ("assembling-machine-2", 0.75)
      ]
  , moduleEffects = ModuleSettings
      { speedBonus  = 0.3
      , productivityBonus = 0.1
      }
  }

main :: IO ()
main = do
  let req  = Requirement "electronic-circuit" 2
      plan = calculatePlan myConfig req
  print plan
```

---

以上のように、

1. **純関数呼び出し**
2. **DSL**
3. **バッチ処理**
4. **CLI ツール**
5. **カスタム設定の差し替え**

といったスタイルで、ユーザーがニーズや好みに合わせて自由に呼び出せるイメージを持っていただければと思います。これをベースに、さらにグラフ出力や JSON エクスポート、Web UI 連携などを拡張していくのも自然な流れです。
