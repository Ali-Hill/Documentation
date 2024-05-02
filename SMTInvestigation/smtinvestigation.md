# Report on the utilisation of SMT to test Plutus Haskell Libraries 

## Introduction 

A quick investigation has been done to determine the suitability of using smt-based libraries to test and verify peoperties of plutus libraries. At this current stage no actual library functions have had additional tests added to them. The work has added these libraries as a dependency and added tests about some example functions. 

The initial investigation has been done in this [repo](plutus-library-testing).

## [Liquid Haskell](https://ucsd-progsys.github.io/liquidhaskell/)

Installing Liquid Haskell was pretty difficult. There were many issues with dependencies and I could not get it to work at all for GHC 9.6.4. Downgrading the library to GHC 9.6.3 and adding tagged versions of `liquidhaskell` and `liquidfixpoint` was needed to get the library installed. 


Actually using Liquid Haskell to write properties that we want to ensure for functions was very nice. Examples can be found [here](https://github.com/Ali-Hill/plutus-library-testing/blob/sbv-test/plutus-tx/test/Liquid.hs). It integrates with haskell language server to automatically check the types within an editor. Also the type annotations apply to all functions. Take the following divide example: 

```haskell
{-@ type NonZero = {v:Int | v /= 0} @-}

{-@ divide :: Int -> NonZero -> Int @-}
divide     :: Int -> Int -> Int
divide n 0 = die "divide by zero"
divide n d = n `div` d

avg       :: [Int] -> Int
avg xs    = if n == 0 then 0 else divide total n
  where
    total = sum xs
    n     = length xs
```

In this above by adding the annotation that the divide function is annotated to require that the denominator has to be nonzero. Any function that uses divide, such as `avg`, is then automatically checked that all calls divide satisfy the annotation. This is also checked automatically in haskell language server. As someone who is familiar with using dependent types, using annotations is an intuitive way of adding properties to functions. This method feels cleaner than alternative as a developer does not need to actually change the functions themselves. See the divide example in the `sbv` section for comparison. 

Trying to use liquid haskell with the actual plutus-tx library posed many issues. Due to the nice integration of the annotations many requirements seem to be placed on these libraries. In trying to get to the point testing the `Ratio` library in plutus-tx many files had to be edited (see this [commit](https://github.com/Ali-Hill/plutus-library-testing/commit/589ca022a2fae3dde783780c0275575ce22ccd9e). Unfortunately I could not fix the final error in `Ratio.hs`: `**** LIQUID: ERROR: Sorry, unexpected panic in liquid-fixpoint! ***************`. 

## SBV[https://github.com/LeventErkok/sbv](https://github.com/LeventErkok/sbv)

Installing SBV was as simple as just adding `sbv` to the `plutus-tx.cabal` to add it to the library. 

Using SBV to verify properties about programs does not use annotations. Instead you have to use SBV's symbolic version of types and implement functions that contain assertions. Full code for the example can be found [here](https://github.com/Ali-Hill/plutus-library-testing/blob/sbv-test/plutus-tx/test/Liquid.hs). Example:

```haskell 
checkedDiv :: (?loc :: CallStack) => SInt32 -> SInt32 -> SInt32
checkedDiv x y = sAssert (Just ?loc)
                         "Divisor should not be 0"
                         (y ./= 0)
                         (x `sDiv` y)

safeCallDiv :: SInt32 -> SInt32 -> SInt32
safeCallDiv x y = ite (y .== 0) 0 (checkedDiv x y)

test3 :: IO [SafeResult]
test3 = safe safeCallDiv
```

The above code implements a new divide function that asserts that the denominator must not be zero. Additional tests can then be run that check that a function is safe given the assertions (see `test3`). Compared to liquid haskell this is less nice in my opinion as users have to actually define and run these tests and check the results. In comparison, liquid haskell will give you a compilation error if you define a function that does not satisfy the assertions. 

There were no issues with compatibility with `sbv` which I assume is because it works with its own symbolic types and does not have the same levels of integration. Actually proving properties about library functions will require redefining and writing tests for all library functions that we care about. This is probably not too much work as long as there are already equivlaent symbolic types. I would worry about whether the symbolic version of the program is equivalent to the normal implementation.

## Conclusion 

Liquid haskell provides a much nicer way to interact and prove properties about haskell programs but has many compatibility issues which may make is unusable for many libraries. SBV had no compatibility issues however but requires reimplementing functions to use its symbolic types. 

