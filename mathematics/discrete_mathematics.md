<link rel="stylesheet" type="text/css" href="../css/style.css">  

# Basic Structures: Sets, Functions, Sequences, Sums and Matrices

## Sets

## Set Operations

## Functions

<TT>Introductions</TT>

<DEF>DEFINITION 1. Let A and B be nonempty sets. A `function f` from A to B is an assignment of exactly one element of B to each element of A. We write $f(a) = b$ if b is the unique element of B assigned by the function f to the element a of A. If $f$ is a function from A to B, we write $f : A \longrightarrow B$</DEF>.  
<BIG>Remark: </BIG> Functions are sometimes also called `mappings` or `transformations`.  

<DEF>DEFINITION 2. If $f$ is a function from A to B, we say that A is the `domain` of $f$ and B is the `codomain` of $f$. If $f(a) = b$, we say that b is the `image` of a and a is a `preimage` of b. The `range`, or `image`, of $f$ is the set of all images of elements of A. Also, if $f$ is a function from A to B, we say that $f$ maps A to B.  

<DEF>DEFINITION 3. Let $f_1$ and $f_2$ be functions from A to R. Then $f_1 + f_2$ and $f_1f_2$ are also functions from A to R defined for all $x \in A$ by:  
$(f_1 + f_2)(x) = f_1(x) + f_2(x)$,  
$(f_1f_2)(x) = f_1(x)f_2(x)$ </DEF>  

<DEF>DEFINITION 4. Let $f$ be a function from A to B and let S be a subset of A. The `image` of S under the function $f$ is the subset of B that consists of the images of the elements of S. We denote the image of S by $f(S)$, so  
$f(S) = \{ t | \exists s \in S (t = f(s))\}.$  
We also use the shorthand $\{f(s) | s \in S\}$ to denote this set.</DEF>  

<TT>One-to-One and Onto Functions</TT>

<DEF>DEFINITION 5. A function $f$ is said to be `one-to-one`, or an `injunction`, iff $f(a) = f(b)$ implies that $a = b$ for all a and b in the domain of $f$. A function is said to be `injective` if it is one-to-one.</DEF>  
<BIG>Remark:</BIG> We can express that $f$ is one-to-one using quantifiers as $\forall a \forall b(f(a) = f(b) \rightarrow a = b)$ or equivalently $\forall a \forall b (a \cancel{=} b \rightarrow f(a) \cancel{=} f(b))$, where the universe of discourse is the domain of the funciton.  
严格递增或者递减的函数一定是`one-to-one`.  

<DEF>DEFINITION 7. A function $f$ from A to B is called `onto`, or a `surjection`, iff $\forall b \in B$ there is an element $a \in A$ with $f(a) = b$. A function $f$ is called `surjetive` if it is onto.</DEF>  

<BIG>Remark:</BIG> A function $f$ is onto if $\forall y \exist x (f(x) = y)$, where the domain for x if the domain of the function and the domain for y is the codomain of the function.  

<DEF>DEFINITION 8. The function $f$ is a `one-to-one correspondence`, or a `bijection`, if it is both one-to-one and onto. We also say that such a function is `bijective`</DEF>.  

<TT>Inverse Functions and Compositions of Functions</TT>  
<DEF>DEFINITION 9. Let $f$ be a one-to-one correspondence from the set A to the set B. The `inverse function` of $f$ is the function that assigns to an element b belonging to B the unique element a in A such that $f(a) = b$. The inverse function of $f$ is denoted by $f^{-1}$. Hense, $f^{-1}(b) = a$ when $f(a) = b$</DEF>.  
If a function f is not one-to-one correspondence, either it is not one-to-one or it is not onto.  
If $f$ is not `one-to-one`, some element b in the codomain is the image of more than one element in the domain.  
If $f$ is not onto, for some element b in the codomain, no element a in the domain exists for which $f(a) = b$.  

<DEF>DEFINITION 10. composition of the functions: $g$ is a function from set A to B, and $f$ is a function from set B to C, then $f \circ g$ is defined:  
$(f \circ g)(a) = f(f(a)).$  </DEF>  

<TT>The Graphs of Functions</TT>  
<DEF>DEFINITION 11. Let $f$ be a function from the set A to the set B. The `graph` of the function $f$ is the set of ordered pairs $\{(a,b) \vert a \in A\enspace and\enspace f(a) = b\}$</DEF>  

<TT>Some Important Functions</TT>

上界和下界函数

<TT>Partial Functions</TT>

<DEF>DEFINITION 13. A `partial function` $f$ from a set A to a set B is an assignment to each element a in a subset of A, called the `domain of definition` of $f$, of a unique element b in B. The sets A and B are called the `domain` and `codomain` of $f$, respectively. We say that $f$ is `undefined` for elements in A that are not in the domain of definition of $f$. When the domain of definition of $f$ equals A, we say that $f$ is a `total order`.</DEF>  
****
# Relations

## Relations and Their Properties

<TT>二元关系</TT>  

<DEF>DEFINITION 1. Let A and B be sets. A `binary relation from A to B` is a subset of $A \times B$</DEF>

A binary relation from A to B is a set R of ordered pairs where 第一个元素从A中来,第二个从B中来  
$a\enspace R\enspace b$ means $(a,b) \in R$  
当$(a,b) \in R$ 那么我们说a is related to b by R  

<DEF>DEFINITION 2. A `relation on a set` A is a relation from A to A</DEF>  
A relation on a set A is a subset of $A \times A$  

## Properties of Relations

<TT>自反性</TT>  

<DEF>DEFINITION 3. A Relation R on a set A is called `reflexive` if $(a,a) \in R$ for every element $a \in A.$</DEF>  

如果说一个集合是自反的那么每个元素都related to itself.  
比如:R是一个包含拥有相同父亲和母亲的人的集合,元组为(x,y),那么这个集合是自反的,因为(x,x)一定符合这个条件.  

<TT>对称性</TT>  

<DEF>DEFINITION 4. A relation R on a set A is called `symmetric` if $(b,a) \in R$ whenever $(a,b) \in R$, for all $a,b \in A$. A relation R on a set A such that for all $a,b \in A$, if $(a,b) \in R$ and $(b,a) \in R$, then $a = b$ is called `antisymmetric`.  </DEF>

<TT>传递性</TT>  

<DEF>DEFINITION 5. A relation R on a set A is called `transitive` whenever $(a,b) \in R$ and $(b,c) \in R$, then $(a,c) \in R$, for all $a,b,c \in A$.</DEF>  

<TT>Composite</TT>  

<DEF>DEFINITION 6. Let R be a relation from a set A to a set B and S a relation from B to a set C. The `composite` of R and S is the relation consisting of ordered pairs $(a,c)$, where $a \in A, c \in C$, and for which there exists an element $b \in B$ such that $(a,b) \in R$ and $(b,c) \in S$. We denote the composite of R and S by $S \circ R$</DEF>

<DEF>DEFINITION A relation on a set A is called an `equivalence relation` if it is reflexive, symmetric, and transitive.</DEF>

<TT>Partial Orderings</TT>  

<DEF>DEFINITION A relation R on a set S is called `partial ordering` or `partial order` if it is reflexive, antisymmetric, and transitive.</DEF>  
<DEF>A set S together with a partial ordering R is called  a `pritially ordered set`, or `poset`, and is denoted by $(S,R)$. Members of S are called `elements` of the poset.</DEF>  

Relation "greater than or equal" $(\ge)$ is a partial ordering on the set of intergers.  
Because $a \ge a$ for every integer $a$, $\ge$ is reflexive. If $a \ge b$ and $b \ge a$, then $a = b$. Hence, $\ge$ is antisymmetric. Finally, $\ge$ is transitive because $a \ge b$ and $b \ge c$ imply that $a \ge c$. It follows that $\ge$ is a partial ordering on the set of integers and $(Z,\ge)$ is a poset.  

<BIG>The notation $a \curlyeqprec b$ is used to denote that $(a,b) \in R$ in an arbitray poset $(S,R)$.</BIG>  

<TT>Comparable</TT>  

When a and b are elements of the poset $(S, \curlyeqprec )$, a 和 b并不是一定存在 $a \curlyeqprec b$ 或者 $a \curlyeqsucc b$ 关系.  
比如整除关系$(Z^+, |)$, 2 is not related to 3, and 3 is not related to 2, because <BIG>$\large 2\enspace \cancel|\enspace 3$ and $\large 3\enspace \cancel |\enspace 2$</BIG>  

<DEF>DEFINITION The elements a and b of a poset $(S, \curlyeqprec)$ are called `comparable` if either $a \curlyeqprec b\enspace or\enspace b \curlyeqprec a$. When a and b are elements of S such that neither $a \curlyeqprec b\enspace nor\enspace b \curlyeqprec a$, a and b are called `incomparable`.</DEF>  

The adjective `partial` is used to describe partial orderings because pairs of elements may incomparable.  
<TT>Total Orderings</TT>  

<DEF>DEFINITION If $(S, \curlyeqprec)$ is a poset and every two elements of S are comparable, S is called a `totally ordered` or `linearly ordered set`, and $\curlyeqprec$ is called a `total order` or a `linear order`. A totally ordered set is also called a `chain`</DEF>

<BIG>EXAMPLE:</BIG> The poset $(Z,\le)$ is totally ordered, because $a \le b\enspace or\enspace b \le a$ whenever a and b are integers.  
The poset $(Z^+,|)$ is not totally ordered. 因为不是任何两个正整数都具有整除关系.

<TT>Well Ordered Set</TT>  

<DEF>DEFINITION $(S, \curlyeqprec)$ is `well-ordered set` if it is a poset such that $\curlyeqprec$ is a total ordering and every nonempty subset of S has a least element.  </DEF>
