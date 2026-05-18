---
layout: default
title: Sean Tang
---

# Partial Evaluation

Below are my notes for the Programming Language Foundations Partial Evaluation Chapter. Download exercise from [here](./assets/misc/PE.v).

An example the textbook gives for constant folding is:
```
X := 3; Y := X + 1 -> X := 3; Y := 4
```

## Generalizing Constant Folding

A partial state is a mapping from string (var name) to option nat (nat is int, and option means optional)

The full definition is:
```
Definition pe_state := list (string * nat)
```

We use this definition because we may want to compare between two partial states and see how they differ. The comparison
is useful for things like handling conditional control flow.

We then write a lookup function which looks through the list; the first occurrence wins.

```
Fixpoint pe_lookup (pe_st : pe_state) (V:string) : option nat :=
  match pe_st with
  | [] ⇒ None
  | (V',n')::pe_st ⇒ if String.eqb V V' then Some n'
                      else pe_lookup pe_st V
  end.
```

The empty pe_state is just an empty list. It will by definition map to None. 

The tactic `compare V V'` substitutes V for V' in the entire statement. 

```
Tactic Notation "compare" ident(i) ident(j) :=
  let H := fresh "Heq" i j in
  destruct (String.eqb_spec i j) as [H|H];
  [ subst j | ].
```

And then prove if pe_lookup succeeds on pe_st for some V and returns n, then V is in pe_st:

```
Theorem pe_domain: ∀ pe_st V n,
  pe_lookup pe_st V = Some n →
  In V (map (@fst _ _) pe_st).
Proof. intros pe_st V n H. induction pe_st as [| [V' n'] pe_st].
  - (*  *) inversion H.
  - (* :: *) simpl in H. simpl. compare V V'; auto. Qed.
```

After that, we're provided with some code to bridge from logical membership (`In`, a proposition)
to boolean membership (`inb`, a computation).

The lemma
```
Lemma inbP : forall A : Type, forall eqb : A->A->bool,
  (forall a1 a2, reflect (a1 = a2) (eqb a1 a2)) ->
  forall a l, reflect (In a l) (inb eqb a l).
```
says: if your equality test `eqb` is correct, then `inb` is an exact boolean test for list membership.

Python proxy:
```python
def inb(a, l):
    return a in l
```

So the overall flow is:
1. Define a partial state and lookup (`pe_state`, `pe_lookup`).
2. Prove a domain property (`pe_domain`): successful lookup implies key is in the state.
3. Use `inbP`-style lemmas to move cleanly between executable boolean checks and logical proofs.

## Arithmetic Expressions

We then define arithmetic partial evaluation:

```
Fixpoint pe_aexp (pe_st : pe_state) (a : aexp) : aexp :=
  match a with
  | ANum n ⇒ ANum n
  | AId i ⇒ match pe_lookup pe_st i with (* <----- NEW *)
             | Some n ⇒ ANum n
             | None ⇒ AId i
             end
  | <{ a1 + a2 }> ⇒
      match (pe_aexp pe_st a1, pe_aexp pe_st a2) with
      | (ANum n1, ANum n2) ⇒ ANum (n1 + n2)
      | (a1', a2') ⇒ <{ a1' + a2' }>
      end
  | <{ a1 - a2 }> ⇒
      match (pe_aexp pe_st a1, pe_aexp pe_st a2) with
      | (ANum n1, ANum n2) ⇒ ANum (n1 - n2)
      | (a1', a2') ⇒ <{ a1' - a2' }>
      end
  | <{ a1 * a2 }> ⇒
      match (pe_aexp pe_st a1, pe_aexp pe_st a2) with
      | (ANum n1, ANum n2) ⇒ ANum (n1 * n2)
      | (a1', a2') ⇒ <{ a1' * a2' }>
      end
  end.
```

Note that we have an extra case where, if `pe_lookup` finds a value for a variable, we replace it with `ANum n`; otherwise we leave it as `AId i`.

The rest we can see supports doing some kinds of evaluation if both terms are ANums. 

An important thing to note is that this evaluator folds constants but does not apply associativity rewrites.

```
Example test_pe_aexp1:
  pe_aexp [(X,3)] <{X + 1 + Y}>
  = <{4 + Y}>.

Example text_pe_aexp2:
  pe_aexp [(Y,3)] <{X + 1 + Y}>
  = <{X + 1 + 3}>.
```

Now we move on to defining weak correctness. The definition the book uses is for weak correctness is:

Where consistent means that for every variable where pe_st assigns a value, st assigns the variable the same value,
whenever a full state st:state is consistent with a partial state pe_st:pe_state,
then evaluating the original `a` is same as evaluating `pe_aexp  pe_st a`. 

More concretely:

def of Consistent
```
Definition pe_consistent (st:state) (pe_st:pe_state) :=
  ∀ V n, Some n = pe_lookup pe_st V → st V = n.
```

def of weak correctness.
"For all `state st` and `pe_state pe_st`, `pe_consistent st pe_st` implies: for all `a`, `aeval st a = aeval st (pe_aexp pe_st a)`."
In plainer terms:
- If the state is consistent with the partial state in that sense that all their variable mappings align,
then for all arith-expressions the result of evaluation should be equal between starting from the run-time state compared to starting from the run-time state on a post partial evaluated version of the arith-expression.

Basically, If the partial-eval table is consistent with the real state table, then before partial-eval and post-partial-eval, both version of code should give same result given same starting state.

```
Theorem pe_aexp_correct_weak: ∀ st pe_st, pe_consistent st pe_st →
  ∀ a, aeval st a = aeval st (pe_aexp pe_st a).

Proof. unfold pe_consistent. intros st pe_st H a.
  induction a; simpl;
    try reflexivity;
    try (destruct (pe_aexp pe_st a1);
         destruct (pe_aexp pe_st a2);
         rewrite IHa1; rewrite IHa2; reflexivity).
  (* Compared to fold_constants_aexp_sound,
     the only interesting case is AId *)
  - (* AId *)
    remember (pe_lookup pe_st x) as l. destruct l.
    + (* Some *) rewrite H with (n:=n) by apply Heql. reflexivity.
    + (* None *) reflexivity.
Qed.
```

As an example of why this is weak correctness, consider the below example:

Eventually, the partial evaluator will also be capable of removing assignments. Ie:

```
  X := 3; Y := X - Y; X := 4
    to
  Y := 3 - Y; X := 4
```

The key step is partial-evaluating the RHS of `Y := …` **after** `X := 3` but **before** `X := 4`. At that point the partial state is `[(X, 3)]`, so we want

```
pe_aexp [(X, 3)] <{ X - Y }>  =  <{ 3 - Y }>
```

and **not** to leave `<{ X - Y }>` in the residual program.

Focus on that arithmetic subproblem. Let `pe_st = [(X, 3)]` and `a = <{ X - Y }>`. Suppose the full state `st` is consistent with `pe_st` (so `st X = 3`). Then:

| Residual expression `pe_aexp pe_st a` | `aeval st (pe_aexp pe_st a)` when `st X = 3` |
|--------------------------------------|-----------------------------------------------|
| `<{ X - Y }>` (no substitution)      | `3 - st Y`                                    |
| `<{ 3 - Y }>` (substitute `X`)       | `3 - st Y`                                    |

Both residuals satisfy **weak** correctness, because the theorem only requires equality with `aeval st a` **in the same `st`**. While `st` still has `X = 3`, reading `X` in the residual and reading the constant `3` happen to agree.

That is exactly why the property is weak: it does **not** rule out a “do nothing” partial evaluator. If we defined `pe_aexp pe_st a = a` for all `pe_st` and `a`, then `pe_aexp_correct_weak` would still be provable (trivially). Weak correctness says the optimized expression is **numerically equivalent in the current state**, not that it is the **right** residual for later commands whose meaning depends on which variables are still unknown.

The bad command transform makes the same point at program level. With `pe_st = [(X, 3)]` after the first assignment:

- **Good residual:** `Y := 3 - Y; X := 4` — uses the fact that `X` was `3` at the assignment to `Y`, then updates `X` to `4`.
- **Bad residual:** `Y := X - Y; X := 4` — still mentions `X` on the RHS of `Y := …`. If we ran that program starting from a state where `st X = 3`, the first assignment to `Y` would still compute `3 - st Y`, so weak correctness of the *expression* `X - Y` vs `3 - Y` can still hold **at that moment**. But after `X := 4`, later reads of `X` differ; the bad program is wrong as a **whole** transformation even though a one-shot arithmetic check in the old state looked fine.

So weak correctness is necessary but not sufficient for partial evaluation of commands: we need a stronger statement that ties residuals to the partial state in a way that survives when parts of `st` are intentionally “forgotten” or overridden.

There is also a stronger correctness theorem. 

First, we define `pe_update` which updates the `st` map with `pe_st`. 

We define `pe_update_correct` with:
For all state, partial state, variables, 
The value of v0 after applying pe_update on st from pe_st is
the same as pe_lookup on pe_st for v0, otherwise it is 
st v0. 

```
Theorem pe_update_correct: ∀ st pe_st V0,
  pe_update st pe_st V0 =
  match pe_lookup pe_st V0 with
  | Some n => n
  | None => st V0
  end.
```

We can also relate `pe_consistent` to `pe_update` in two ways:
- pe_update_consistent: `For all st pe_st, pe_consistent (pe_update st pe_st) pe_st`. In plain english, for all st pe_st, the state as a result of updating st with pe_st is consistent with pe_st, where consistent means that for every variable where pe_st assigns a value, st assigns the variable the same value
- pe_consistent_update: `∀ st pe_st,   pe_consistent st pe_st → ∀ V, st V = pe_update st pe_st V`. In plain english, for all st pe_st, pe_consistent between st and pe_st implies that for all Var v, the value v maps to in st is same as the value v maps to after applying update on st with pe_st.

Thus, the final definition of pe_aexp_correct (strong) is:
```
Theorem pe_aexp_correct: ∀ (pe_st:pe_state) (a:aexp) (st:state),
  aeval (pe_update st pe_st) a = aeval st (pe_aexp pe_st a).
```
Or in plain english, 
- for all partial eval state pe_st, arithmetic expression a, and state st,
- running arithmetic evaluation on expression a from state `pe_update st pe_st`
- should result in the same as 
- running arithmetic evaluation on the partially-evaluated expression from the original state st.

Intuitively, running a program using partial evaluation is a two-stage process. In the first, static stage, we partially evaluate with respect to some partial state to get a residual expression/program. In the second, dynamic stage, we evaluate the residual form with respect to the rest of the state. This dynamic state provides values for variables unknown in the static (partial) state.

## Boolean Expression

Since this language has no boolean variables, we call
pe_aexp where appropriate to call pe_lookup and sub it out
for values. 

```
Fixpoint pe_bexp (pe_st : pe_state) (b : bexp) : bexp :=
  match b with
  | <{ true }> ⇒ <{ true }>
  | <{ false }> ⇒ <{ false }>
  | <{ a1 = a2 }> ⇒
      match (pe_aexp pe_st a1, pe_aexp pe_st a2) with
      | (ANum n1, ANum n2) ⇒ if n1 =? n2 then <{ true }> else <{ false }>
      | (a1', a2') ⇒ <{ a1' = a2' }>
      end
  | <{ a1 <> a2 }> ⇒
      match (pe_aexp pe_st a1, pe_aexp pe_st a2) with
      | (ANum n1, ANum n2) ⇒ if negb (n1 =? n2) then <{ true }> else <{ false }>
      | (a1', a2') ⇒ <{ a1' <> a2' }>
      end
  | <{ a1 <= a2 }> ⇒
      match (pe_aexp pe_st a1, pe_aexp pe_st a2) with
      | (ANum n1, ANum n2) ⇒ if n1 <=? n2 then <{ true }> else <{ false }>
      | (a1', a2') ⇒ <{ a1' <= a2' }>
      end
  | <{ a1 > a2 }> ⇒
      match (pe_aexp pe_st a1, pe_aexp pe_st a2) with
      | (ANum n1, ANum n2) ⇒ if n1 <=? n2 then <{ false }> else <{ true }>
      | (a1', a2') ⇒ <{ a1' > a2' }>
      end
  | <{ ~ b1 }> ⇒
      match (pe_bexp pe_st b1) with
      | <{ true }> ⇒ <{ false }>
      | <{ false }> ⇒ <{ true }>
      | b1' ⇒ <{ ~ b1' }>
      end
  | <{ b1 && b2 }> ⇒
      match (pe_bexp pe_st b1, pe_bexp pe_st b2) with
      | (<{ true }>, <{ true }>) ⇒ <{ true }>
      | (<{ true }>, <{ false }>) ⇒ <{ false }>
      | (<{ false }>, <{ true }>) ⇒ <{ false }>
      | (<{ false }>, <{ false }>) ⇒ <{ false }>
      | (b1', b2') ⇒ <{ b1' && b2' }>
      end
  end.
```

From that point of view, the correctness definition is also similarly defined:

in plain english, 
- for all partial eval state pe_st, bool-expression a, state st, 
- running *bool-eval* on bool-expression a starting from the state state st updated by partial eval state pe_st 
- should result in the same as 
- running *bool-eval* on bool-expression after partial eval starting from the original state st. 

```
Theorem pe_bexp_correct: ∀ (pe_st:pe_state) (b:bexp) (st:state),
  beval (pe_update st pe_st) b = beval st (pe_bexp pe_st b).
Proof.
  intros pe_st b st.
  induction b; simpl;
    try reflexivity;
    try (remember (pe_aexp pe_st a1) as a1';
         remember (pe_aexp pe_st a2) as a2';
         assert (H1: aeval (pe_update st pe_st) a1 = aeval st a1');
         assert (H2: aeval (pe_update st pe_st) a2 = aeval st a2');
           try (subst; apply pe_aexp_correct);
         destruct a1'; destruct a2'; rewrite H1; rewrite H2;
         simpl; try destruct (n =? n0);
         try destruct (n <=? n0); reflexivity);
    try (destruct (pe_bexp pe_st b); rewrite IHb; reflexivity);
    try (destruct (pe_bexp pe_st b1);
         destruct (pe_bexp pe_st b2);
         rewrite IHb1; rewrite IHb2; reflexivity).
Qed.
```

# Partial Evaluation of Commands, without loops

Partial Eval of commands is similar to partial eval of aexp or bexp, where we return a residual aexp/bexp/com after applying partial eval. 

Partial Evaluator commander may also not terminate on all commands (? wouldn't that be dangerous). Ideally, we build one that terminates on all commands, but also does optimization. The techniques of adjusting how it terminates is known as "binding-time improvement", where binding-time is the time when variable value is known. 

Here, they define a notation 

```
c1 / st ==> c1' / st'
```

where c1 is an initial command, and st is an inital partial state; after partiail eval, we reach a final c1' command post optimization, and st' as final partial state.

## Assignment

The book first considers assignment. We start by replace constant assignments with residual `skip` and adding it to the partial state table. 

Since as we move down a list of commands, we may need to remove/rebind certain variable-values, we introduce `pe_remove` and `pe_add`. 

`pe_remove` is for the dynamic assignment - it removing variable which should be dynamically assigned during run time. `pe_add` is for static assignments, which is adding variable which can be statically assigned/propagated during partial evaluation. 

What is the difference between `pe_update` and `pe_add`?

`pe_add` adds a single new variable, while `pe_update` the state table with an entire partial-state. 

The implementation for `pe_remove` constructs a new list by selectively including elements that do not match the removal key. 

The correctness of `pe_remove` is defined in terms of pe_lookup returning None fo the removed key and returning the original pe_look result if it is not removed key.

The implementation for `pe_add` removes the V from the pe_st and then adds the new (V, n). 

The correctness of `pe_add` is defined as for all pe_st V n V0, the added variable-value pair is the first one returned by pe_lookup on the post added state. 

## Conditional

For conditionals where we cannot determine the branching direction, we evaluate both branches and then take the intersection to propagate down the further commands. 

Take this example:

```
      X := 3;
      if Y ≤ 4 then
          Y := 4;
          if X ≤ Y then Y := 999 else skip end
      else skip end
```

gets optimized to

```
      skip;
      if Y ≤ 4 then
          skip;
          skip;
          Y := 4
      else skip end
```

The inner if was optimized out because we can determine the conditional statically. We thus drop the if and replace it with a skip. However, since Y:=4 is not in the intersection between the Y<=4 and else branch, we need to remove it from the partial state and add it back to the branch for dynamic assignment. 

To implement this, we first implement `pe_disagree` which computes for some variable if there is discrepancy between two different states. We then prove the property that `pe_disagree` can only be true if a variable appear in at least one of them.  

```
Definition pe_disagree_at (pe_st1 pe_st2 : pe_state) (V:string) : bool :=
  match pe_lookup pe_st1 V, pe_lookup pe_st2 V with
  | Some x, Some y ⇒ negb (x =? y)
  | None, None ⇒ false
  | _, _ ⇒ true
  end.
Theorem pe_disagree_domain: ∀ (pe_st1 pe_st2 : pe_state) (V:string),
  true = pe_disagree_at pe_st1 pe_st2 V →
  In V (map (@fst _ _) pe_st1 ++ map (@fst _ _) pe_st2).
Proof. unfold pe_disagree_at. intros pe_st1 pe_st2 V H.
  apply in_app_iff.
  remember (pe_lookup pe_st1 V) as lookup1.
  destruct lookup1 as [n1|]. left. apply pe_domain with n1. auto.
  remember (pe_lookup pe_st2 V) as lookup2.
  destruct lookup2 as [n2|]. right. apply pe_domain with n2. auto.
  inversion H. Qed.
```

Furthermore, we implement `pe_compare` which returns the list of variables that disagree. 

the intersection is then defined in terms of `pe_compare_removes`, which removes from one list all of the dis-agreeing variables. 

Finally, an `assign` function is defined which generates a sequence of assignment states based on a list of ids and a pe_st which provides the mapping from id to value. 

## The Partial Evaluation Relation

Finally with the helpers in place, we define partial evaluation relation as an inductive relation. Here, we define the specification of valid cases of transitioning from c / st -> c' / st'

Using the special notation from above, we simplify the syntax 

```
  where "c1 '/' st '==>' c1' '/' st'" := (pe_com c1 st c1' st').
```

Now we can look at different kinds of commands and what are the valid transformations for them:

1. PE_Skip
skip / pe_st ⇒ skip / pe_st
Nothing to do.

2. PE_AsgnStatic
If pe_aexp( pe_st a1 ) is <{ n1 }> then we can replace the assignment with a   <{skip}> and use pe_add to add the variable-value to the partial-eval state table.  

3. PE_AsgnDynamic
If pe_aexp( pe_st a1 ) does not evaluate to an actual number, then we need to remove the variable-value pair from the partial-eval state table so we can properly dynamically execute it. 

4. PE_Seq
If the result of partial eval on c1 / st is c1' / st' and partial eval from c2 /st' is c2' / st'', then they can be sequence where <{c1;c2}> / st ==> <{c1';c2'}> / st''.

5. PE_IfTrue / PE_IfFalse
Eliminate branches based on eval result of variable b1 which determines branch. If true, keep c1. If false, keep c2. 

6. PE_If
If we cannot determine if b1 is true or false, then we step do partial eval on c1 and c2 to get c1' and c2' respectively. Then, we define the partial eval result to be:
```
        ==> <{if pe_bexp pe_st b1
             then c1' ; assign pe_st1 (pe_compare pe_st1 pe_st2)
             else c2' ; assign pe_st2 (pe_compare pe_st1 pe_st2) end}>
            / pe_removes pe_st1 (pe_compare pe_st1 pe_st2)
```
Note how we take result of partial-eval on c1, c1', and then post-pend the dis-agreeing variables assignments. Same for c2. Then, we do clean up and only keep the intersection (which is done by removing the dis-agreeing variables).

