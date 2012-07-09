# TOC

1. What is it?  
    DEMO
2. Technical details
3. How to?  
    3.1 Generate your own PLT  
    3.2 Use your own PLT  
    3.3 See some results on the command line  
4. Weak points and future solutions

# 1. WHAT IS IT?

Have you ever tried Hoogle? http://www.haskell.org/hoogle/
 
If you haven't, here's a brief description. 

Quite often you don't know the name of a function, or the name of the module it resides in, but you know what types you'd like this function to accept and what types this function will output.

Hoogle let's you search based on type signatures of functions. One small problem though... It's Haskell only, no Erlang.

Wouldn't it be nice to type in something like

       proplists:?(any(), list()) -> any()

and get back

       proplists:get_value

Or type in 

       "(erlang:timestamp(),erlang:timestamp())->integer()"

and get back

       erlang:now_diff

?

That's the goal of HINT: to provide you with a way to search for function definitions and documentation by function type signatures.

# DEMO

1. Start web.sh
2. Go to http://127.0.0.1/52012

Give some time for web.sh to start completely. If it doesn't work, clean && compile && run.

3. Demo URLs (on Monday):  
- http://memorici.de:52012
- http://booktu.com

One of these will work :)




# 4. HOW TO

## 4.1 Generate your own PLT

The problem with PLTs is that they are slow to generate and that they contain hard-coded links to .beam files on you system, so we cannot provide you with one.

We recommend you generate a "tiny version" of PLT. Otherwise you might end up waiting for search results forever (or run out of memory).

In order to generate a tiny PLT, run the following in your shell:
       dialyzer --build_plt --apps kernel stdlib erts crypto sasl --output_plt tiny_plt

## 4.2 Use your own PLT

Make sure that application environment contains

           {env, [{plt_path, PATH/TO/YOUR/PLT]}
or
           {env, [{plt_path, {priv_dir, PLT}}]}


## 4.3 See some results on the command line

In PROJECT_ROOT:

       bash> erl -pa apps/hint_search/ebin
       > hint_search:start().
       > hint_search:q("(erlang:timestamp(),erlang:timestamp())->integer()").
       {ok,[{{timer,now_diff,2},1.5},
            {{dets,init,2},0.9},
            {{crypto,dh_generate_parameters,2},0.9},
            {{dets,next,2},0.9},
       ...

# 2. Technical details

Frontend:
- Cowboy web-server
- Custom controllers
- Regular web stuff
- Fuzzy search on provided names

Backend:
- derive Module, Funciton, Arity from user request
- create possible permutations of these
- call Dialyzer to check types
- create a ranking of Module-Function pairs based on Dialyzer output
- return these to the user

On startup we read from a supplied PLT file and cache information on all available MFAs in the PLT (plt_cache_server)

Once we receive a request, we do the following:

1. Derive Module, Funciton, Arity
2. For each MFA in cache where A = Arity:
    2.1 For each possible permutation of parameters in MFA  
    2.2 Write custom functions:  

            -spec func_permutation_1(UserProvidedSpecs...
            func_permutation_1(Param1, Param2, Param3, ..., ParamArity) ->
                       M:F(permutation 1 of Param1, Param2, Param3, ..., ParamArity).

            -spec func_permutation_2(UserProvidedSpecs...
            func_permutation_2(Param1, Param2, Param3, ..., ParamArity) ->
                      M:F(permutation 2 of Param1, Param2, Param3, ..., ParamArity).
            ...
3. Once the file with custom functions is complete, run Dialyzer over it
4. For each warning generated by Dialyzer:  
      4.1 Match permutation and warning against MFA  
      4.2 Store warnings with MFA  
5. For each MFA for which there are no warnings (full type match):  
      5.1 Store the MFA with no warnings
6. For each MFA that we have  
      6.1 Rank based on module name (extremely long and extremely short names decrease the rank)  
      6.2 Rank based on function name (extremely long names decrease the rank)  
      6.3 Rank based on warnings  
      - Various diyalizer warnings decrease rank  
      - If there are less warnings than permutations, it means that some of the permutations matched, increase rank  
      - If there are no warnings, all permutations matched, increase rank  
7. Final rank for each MFA is the sum all ranks for that MFA
8. Reverse sort MFAs based on rank
9. Return list of MFAs
   
# 3. Weak points and future solutions

1. Requires PLT  
    1.1 Takes insane amounts of time to build  
    Solution: build several smaller PLTs, search across them  
    1.2 Takes insane amounts of memory for Dialyzer to process   
    Solution:  
    - build several smaller PLTs, search across them cache  
    - search result for common searches  

2. Permutations  
    2.1 Take a long time to generate on each search  
    Solution:  
    - precompute permutations when loading PLT on startup  
    - save files with premutations for more common searches  
