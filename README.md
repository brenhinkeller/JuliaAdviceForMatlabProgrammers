# Julia Advice for MATLAB (or R or Python) programmers

Here are a few things I've learned while transitioning from a mix of MATLAB and C to the [Julia](https://julialang.org) programming langage, starting around 2018. Some of this is rather straightforward, while other points may be less obvious.

The short, narrative, version is that a [multiple dispatch](https://en.wikipedia.org/wiki/Multiple_dispatch)-based programming language like Julia effectively brings with it a whole *dispatch-oriented* programming paradigm, which you have to embrace in order to really get the most out of Julia. Every function in Julia, all the way down to things like `+` (and even, e.g., `Base.add_int` below that) falls into a dispatch hierarchy, which you can freely extend. Julia is not the first language to have this (c.f. [Dylan](https://en.wikipedia.org/wiki/Dylan_(programming_language))), but is one of if not *the* first to combine this with JAOT compilation to avoid runtime performance overhead and enable a remarkable blend of speed and interactivity. This programming paradigm brings with it both some amazing advantages like [composability](https://www.youtube.com/watch?v=kc9HwsxE1OY), but also some new pitfalls to watch out for (particularly, [type-instability](https://www.johnmyleswhite.com/notebook/2013/12/06/writing-type-stable-code-in-julia/), which we will talk about more here below). Meanwhile, some habits and code patterns that may be seen as "best practices" in languages like Matlab, Python, or R can be detrimental and lead to excess allocations in Julia (see, e.g., the sections about "vectorization" below, as well as about different ways of indexing on the RHS of an assignment), so it may almost be easier to switch to Julia and get good performance from day 1 if you are coming from a language like C where you are used to thinking about allocations, in-place methods, and loops being fast.


## The very basics:
There are many different ways of using Julia -- in an IDE (e.g. [Juno](https://junolab.org) or [Julia for VSCode](https://www.julia-vscode.org)), in a Jupyter notebook via [IJulia](https://github.com/JuliaLang/IJulia.jl), in a [Pluto notebook](https://plutojl.org), integrated with a command-line editor like Vim or Emacs, or just directly in the terminal. In any case though, you will likely end up interacting with a Julia *REPL*, where you can type in Julia code and get results back.

#### At the REPL,
* Typing `?` followed by the name of a function (or type, etc.) name will give you help/documentation on that function (or type, etc.).
  * This documentation is usually quite good, but you have to know the exact name of the thing you need. If you don't know that, try the `apropos` function to find the exact names of a few relevant things (try, for example `apropos("sparse array")`). You can also access these suggestions by entering a string rather than raw text at the `?` prompt.
* Typing `]` opens the package manager
* Typing `;` gives you the system command line
* Typing `@less` or `@edit` followed by a function call will show you the source code for that function
* for more advanced REPL tips and tricks, see [this video](https://www.youtube.com/watch?v=EkgCENBFrAY))

#### Some quick-start guides/cheatsheets:
* A handy Matlab-Python-Julia rosetta stone: https://cheatsheets.quantecon.org/
* The ["Noteworthy Differences from other Languages" section](https://docs.julialang.org/en/v1.5/manual/noteworthy-differences/) of the official Julia docs
* "The fast track to Julia": https://juliadocs.github.io/Julia-Cheat-Sheet/

## Common 'gotcha's:
* if `A` is an array, **assigning** `B = A` will copy`A` *by reference* such that both `A` and `B` point to the same memory,  i.e., if you subsequently change `A[1]`, that'll also change `B[1]`. If you don't want this, you need to make a copy, e.g. `B = copy(A)`.
* Slicing an array by **indexing**, with (e.g.)  `x = a[:, 5]` or `a = x[x .> 5]`, etc., *makes a copy*. This is great if you *want* a brand new array, but can be slow if you don't. In the latter case, you can instead use a [`view`](https://docs.julialang.org/en/latest/base/arrays/#Base.view), e.g. `view(a, :, 5)`, which will be much much faster where applicable. 
  * You can turn array-style indexing into a view with the `@views` macro (`@views a[:, 5]` equals `view(a, :, 5)`).
* For **element-wise** operations on an array, you put a dot on _every_ function/operator (not just a some, as in Matlab). 
  * This includes functions you call with `()`. For example, `f.(x)` is different than `f(x)`.
  * This known as **dot-broadcasting**, and can be done for just about any function or operator. 
* Extending the above, `.=` is different than `=`. To use  `.=` the variable on the left of the assignment must already exist and be the right size. If it does though, `.=` will be much more efficient than `=` since it will fill the already-existing array rather than allocating a new one. 
  * If you don't want to write all the dots, you can use the `@.` macro. `@. r  = a * b + sin(c)` is equivalent to `r .= a .* b .+ sin.(c)`. In either case, if you have "dot-broadcasting" for every operation in a line the whole thing will get "fused" with sometimes-significant performance gains.
* For **timing** things (in the context of optimizing performance), you generally want to use `@btime` (from [BenchmarkTools.jl](https://github.com/JuliaCI/BenchmarkTools.jl)), not `@time`, and unless you're timing something that will always be called in global scope you probably want to "interpolate" any global variables into the `@btime` macro with `$`  (e.x. `@btime somefunction($some_array)`)
* For more details, see the excellent articles on [Julia Antipatterns](https://www.oxinabox.net/2020/04/19/Julia-Antipatterns.html) by Lyndon White or [7 Julia Gotchas](http://www.stochasticlifestyle.com/7-julia-gotchas-handle/) by Chris Rackauckas

## Performance:
(see also: https://docs.julialang.org/en/v1.5/manual/performance-tips/)
* **_Loops_** are fast in Julia. It sounds simple but it was hard to really believe / to unlearn my Matlab instincts. In particular, an `@inbounds @simd for` loop doing simple arithmetic operations, array indexing, or really anything that's *type-stable*, should run at C-like or faster speeds.
  * Type-stable, non-allocating loops are just as fast as broadcasting (if not faster!)

* **_Type-stability_**: Use `@code_warntype` followed by a function call to check for type-stability (anywhere it prints a red `Any` is bad). Type-stable code is, like, two orders of magnitude faster than type-unstable code (though even type-unstable code may be faster than Python :p). If the `@code_warntype` output is overwhelming, start by just looking at the list of variables at top and try to make sure they're all green concrete types and not red `Any`s.
  * !However: putting restrictive type assertions on function arguments doesn't make the function faster, and is more useful for dispatch than for anything to do with type stability. Instead, type-stability is all about avoiding cases where instability is introduced in the first place, and if you can't then sanitizing any unstable things (like variables pulled from a `Dict`) with a typeassert (e.g. `x::Array{Float64,1}`) prior to using them (in fact, probably better to avoid `Dict`s altogether except perhaps for interactive use)
  * If you want to "descend" into the functions of your code iteratively to find exactly where type-instability is coming from in the depths of some complicated package, there is the ominously-named [Cthulhu.jl](https://github.com/JuliaDebug/Cthulhu.jl) -- effectively a "type inference debugger".

* **_Vectorization_** is used to refer to two very different concepts in CS, both of which occur in Julia. One is a syntax thing, the other is a hardware thing. The syntax kind is what you know from Matlab (essentially what we discussed as "broadcasting" above), the hardware kind is about the amazing things you can do with your CPU's vector registers / avx extensions (basically like a mini GPU within your CPU), which you can access with [LoopVectorization.jl](https://github.com/chriselrod/LoopVectorization.jl).  
  * You can [LoopVectorization.jl](https://github.com/chriselrod/LoopVectorization.jl) either on a loop with `@avx for`, or on a dot-broadcasted operation with `@avx  @.` You can only use this in cases where the iterations of the loop can be conducted in arbitrary order, since the vector registers will be running the same operations on several iterations of your loop at the same time. 
  * See also [`@simd`](https://docs.julialang.org/en/v1.5/base/base/index.html#Base.SimdLoop.@simd)

* To *follow the compilation pipeline* and see how your Julia code is being translated into intermediate representations, and finally machine code, you can use (e.g., here for the trivial example of `1+1`)
  * `@code_lowered`  Prints Julia SSA-form IR
  ```julia
julia> @code_lowered 1 + 1.0
CodeInfo(
1 ─ %1 = Base.promote(x, y)
│   %2 = Core._apply_iterate(Base.iterate, Base.:+, %1)
└──      return %2
)
  ```
  * `@code_warntype` like `@code_lowered`, but also shows type-inference information
  ```julia
julia> @code_warntype 1 + 1.0
Variables
  #self#::Core.Const(+)
  x::Int64
  y::Float64

Body::Float64
1 ─ %1 = Base.promote(x, y)::Tuple{Float64, Float64}
│   %2 = Core._apply_iterate(Base.iterate, Base.:+, %1)::Float64
└──      return %2
  ```
  * `@code_llvm`     Prints LLVM bitcode
  ```julia
julia> @code_llvm 1 + 1.0
;  @ promotion.jl:321 within `+'
define double @"julia_+_455"(i64 signext %0, double %1) {
top:
; ┌ @ promotion.jl:292 within `promote'
; │┌ @ promotion.jl:269 within `_promote'
; ││┌ @ number.jl:7 within `convert'
; │││┌ @ float.jl:94 within `Float64'
      %2 = sitofp i64 %0 to double
; └└└└
;  @ promotion.jl:321 within `+' @ float.jl:326
  %3 = fadd double %2, %1
;  @ promotion.jl:321 within `+'
  ret double %3
}
  ```
  * `@code_native`   Prints native assembly code
  ```julia
julia> @code_native 1 + 1.0
	.section	__TEXT,__text,regular,pure_instructions
; ┌ @ promotion.jl:321 within `+'
; │┌ @ promotion.jl:292 within `promote'
; ││┌ @ promotion.jl:269 within `_promote'
; │││┌ @ number.jl:7 within `convert'
; ││││┌ @ float.jl:94 within `Float64'
	vcvtsi2sd	%rdi, %xmm1, %xmm1
; │└└└└
; │ @ promotion.jl:321 within `+' @ float.jl:326
	vaddsd	%xmm0, %xmm1, %xmm0
; │ @ promotion.jl:321 within `+'
	retq
	nopw	(%rax,%rax)
; └
  ```

## Other tips:
* In Julia, you generally don't use `printf`. Instead you can interpolate variables into strings with `$` and use plain old `print` (or `println`), ex: "The values varied from $x to $y"
```julia
julia> x = 5; y = 7;
julia> print("The values varied from $x to $y")
The values varied from 5 to 7
```
