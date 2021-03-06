# GrammaticalEvolution

GrammaticalEvolution provides the evolutionary technique of Grammatical Evolution optimization (cite) for Julia. The library focuses on providing the tools necessary to construct a grammar and evolve a population without forcing the user to use a large framework. No XML or configuration files are necessary.

One important aspect of the GrammaticalEvolution library is that uses the strength of Julia to insert arbritrary into the program. Instead of creating a string that then has to be parsed, GrammaticalEvolution instead directly generates Julia code that can then be run. The advantages of doing so are: speeed, simplification of the grammar, and the grammar does not have to be defined in an auxillary file.

## Defining a grammar

The following code defines a grammar:
```julia
@grammar <name> begin
  rule1 = ...
  rule2 = ...
end
```

## Allowed rules

The following rules are supported:
* Terminals: Strings, numbers, and `Expr`s
* Or: `a | b | c`
* And: `a + b + c`
* Grouping: `(a + b) | (c + d)`

# Example
In the `examples` directory there is an example to learn an arbritrary mathemtical equation. Below is an annotated version of that code.

First, we can define the grammar:
```julia
  @grammar example_grammar begin
    start = ex
    ex = number | sum | product | (ex) | value
    sum = Expr(:call, :+, ex, ex)
    product = Expr(:call, :*, ex, ex)
    value = :x | :y | number
    number[convert_number] = digit + '.' + digit
    digit = 0:9
  end
```

Every grammar must define a `start` symbol. This grammar supports equations that add and multiply values together, but it's trivial to build more mathematical operations into the grammar.

There are several items of note:
1. Some rules result in a Julia `Expr`. This results in the Julia code that makes a function call to the selected function.
2. The `number` rule is followed by `[convert_number]`. This appends an action to the rule which is invoked when the rule gets applied.
3. The rule `digit = 0:9` is translated into `digit = 0 | 1 | 2 | .. | 9`

The action `convert_number` is defined as:
```julia
convert_number(lst) = float(join(lst))
```

Next individuals which are generated by the grammar can be defined as:
```julia
type ExampleIndividual <: Individual
  genome::Array{Int64, 1}
  fitness::Float64
  code

  function ExampleIndividual(size::Int64, max_value::Int64)
    genome = rand(1:max_value, size)
    return new(genome, -1.0, nothing)
  end

  ExampleIndividual(genome::Array{Int64, 1}) = new(genome, -1.0, nothing)
end
```

with a population of the individuals defined as:
```julia
type ExamplePopulation <: Population
  individuals::Array{ExampleIndividual, 1}

  function ExamplePopulation(population_size::Int64, genome_size::Int64)
    individuals = Array(ExampleIndividual, 0)
    for i=1:population_size
      push!(individuals, ExampleIndividual(genome_size, 1000))
    end

    return new(individuals)
  end
end
```

Finally, the evaluation function for the individuals can be defined as:
```julia
function evaluate!(grammar::Grammar, ind::ExampleIndividual)
  fitness::Array{Float64, 1} = {}

  # transform the individual's genome into Julia code
  try
    ind.code = transform(grammar, ind)
    @eval fn(x, y) = $(ind.code)
  catch e
    println("exception = $e")
    ind.fitness = Inf
    return
  end

  # evaluate the generated code at multiple points
  for x=0:10
    for y=0:10
      # this the value of the individual's code
      value = fn(x, y)

      # the difference between the ground truth
      diff = (value - gt(x, y)).^2

      if !isnan(diff) && diff > 0
        insert!(fitness, length(fitness)+1, sqrt(diff))
      elseif diff == 0
        insert!(fitness, length(fitness)+1, 0)
      end
    end
  end

  # total fitness is the average over all of the sample points
  ind.fitness = mean(fitness)
end
```

To use our defined grammar and population we can write:
```julia
# here is our ground truth
gt(x, y) = 2*x + 5*y

# create population
pop = ExamplePopulation(500, 100)

fitness = Inf
generation = 1
while fitness > 1.0
  # generate a new population (based off of fitness)
  pop = generate(example_grammar, pop, 0.1, 0.2, 0.2)

  # population is sorted, so first entry it the best
  fitness = pop[1].fitness
  println("generation: $generation, max fitness=$fitness, code=$(pop[1].code)")
  generation += 1
end
```

A couple items of note:
* While `generate` is defined in the base package, it is easy to write your own version.
* Both `Population` and `Individual` are user defined. The fields of these types can differ from what are used in `ExamplePopulation` and `ExampleIndividual`. However, this may require several methods to be defined so the library can index into the population and genome.

The following piece of code in `evaluate!` may look strange:
```julia
ind.code = transform(grammar, ind)
@eval fn(x, y) = $(ind.code)
```

So it's worth explaining it in a little more detail. The first part `ind.code = transform(grammar, ind)` takes the current genome of the genome and turns it into unevalauate Julia code. In the code we use the variables `x` and `y` which are currently not defined.

The variables `x` and `y` are defined in the loop further below. However, they must first be bound to the generated code. Simply running `eval` won't work because it is designed to only use variables defined in the global scope. The line `@eval fn(x, y) = $(ind.code)` creates a function that has its body bound to an evaluated version of the code. When `fn(x, y)` is called, the variables `x` and `y` will now be in scope of the code.


[![Build Status](https://travis-ci.org/abe.schneider@gmail.com/GrammaticalEvolution.jl.svg?branch=master)](https://travis-ci.org/abe.schneider@gmail.com/GrammaticalEvolution.jl)
