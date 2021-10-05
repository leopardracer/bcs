<h1 align="center">ark-bcs tutorial</h1>

In this tutorial, we are going to write a public coin RS-IOP to verify the sum of univariate polynomial.

##### Table of Contents

* [Background](#background)
* [Example Protocol Spec](#example-protocol-spec)
* [Build Simple Univariate Sumcheck](#build-simple-univariate-sumcheck)
* [Build Main Protocol](#build-main-protocol)
* [Put it together: Proof Generation](#put-it-together-proof-generation)
* [Bonus: Debug Your IOP Code using iop_trace!](#bonus-debug-your-iop-code-using-iop_trace)
* [Bonus: Write R1CS Constraints for the Verifier](#bonus-write-r1cs-constraints-for-the-verifier)
* [Further Reading: Multi-round example](#further-reading-multi-round-example)

## Background

In `ark-bcs`, a public coin (RS-)IOP have two phases: 

- **Commit Phase:** Prover run the interactive proof generation code, where prover can send message oracles with or without degree bound, and short messages. Verifier can send message sampled uniformly without reading prover messages. Ideally, verifier delays all verifier messages to next phase. 
- **Query and decision phase:** Verifier starts in a clean state, can query prover's message in arbitrary order, and have access to all previously sent messages. Verifier will generate an user-defined output. Prover do not have query and decision phase. 

Here is a table summarizing what prover/verifier can do in each phase. 

|                                   | Commit Phase (Prover)         | Commit Phase (Verifier)                                      | Query Phase (Verifier)                               |
| --------------------------------- | ----------------------------- | ------------------------------------------------------------ | ---------------------------------------------------- |
|                                   | `prove`: send prover messages | `register_iop_structure`: simulate the interaction for an (RS-)IOP | `query_and_decide`: query prover messages and decide |
| Previously Sent Prover Messages   | Yes                           | No                                                           | Oracle Access Only                                   |
| Previously Sent Verifier Messages | Yes                           | Yes*                                                         | Yes                                                  |
| Parameter                         | Yes                           | Yes                                                          | Yes                                                  |
| Public Input                      | Yes                           | No**                                                         | Yes                                                  |
| Private Input                     | Yes                           | No                                                           | No                                                   |

**: BCS Paper does not allow verifier to have access to previously sent verifier messages, but our implementation makes this possible for flexibility. Use with caution!*

***:  Although in BCS paper, public input can decide number of rounds, `ark-bcs` does not allow transcript structure (e.g. number of rounds, degree bound, etc) to depend on public input. Instead, user can specify verifier parameter that can change the transcript structure.*

## Example Protocol Spec

Here is the spec for the example protocol we will build!

**Private Input**: Coefficients of`poly0`, `poly1`. *For this example, those two coefficients are selected arbitrarily. In real scenario, those coefficients can come from elsewhere.*

**Public Input:** asserted sum of `poly0` over summation domain, asserted sum of `poly1` over summation domain

**Prover & Parameters:** evaluations domain, summation domain, degree bound

**Satisfy Condition:** asserted sums are correct. 

| Prover*                                                      | Verifier                                  |
| ------------------------------------------------------------ | ----------------------------------------- |
|                                                              | Send two random field elements `r0`, `r1` |
| Send evaluations`r0*poly0`, `r1*poly1`over evaluation domain in one round.** |                                           |
| Invoke univariate sumcheck on `r0*poly0`                     | Invoke univariate sumcheck on `r0*poly0`  |
| Invoke univariate sumcheck on `r1*poly1`                     | Invoke univariate sumcheck on `r1*poly1`  |

**: Disclaimer: This protocol is solely for an example and is not an implementation of any real-world protocol. However, we believe user can easily extend the example to a much more complex one.*

***: In real scenario, we probably do not send this message because same message may already present elsewhere (e.g.as witness assignment), and we can simply take a reference to that message.*

## Build Simple Univariate Sumcheck

Before we write the main protocol, let's build a subprotocol `SimpleSumcheck` so that the main protocol can simply invoke this subprotocol as a black box. Note that this protocol is solely a toy example and not optimized, so do not use it in production!

### Overview of Simple Univariate Sumcheck

In this simple univariate sumcheck, the verifier has oracle access to the evaluations of polynomial`f` over evaluation domain. We want to check if the sum of evaluations of `f` at summation domain equals to `gamma`. 

The sumcheck verifier needs to have `degree`, `evaluation_domain`, `summation_domain` as parameter. So, we define the following struct:

```rust
#[derive(Clone, Debug)]
pub struct SumcheckVerifierParameter<F: PrimeField> {
    /// Degree of the polynomial input
    pub degree: usize,
    /// evaluation domain
    pub evaluation_domain: Radix2EvaluationDomain<F>,
    /// summation domain
    pub summation_domain: Radix2EvaluationDomain<F>,
}
```

Prover's parameter in this protocol also needs the coefficients of the polynomial that we take the sum, and we define the prover parameter is below. In this example, we use `Radix2EvaluationDomain` defined in [`ark-poly`](https://crates.io/crates/ark-poly) library. 

```rust
#[derive(Clone, Debug)]
pub struct SumcheckProverParameter<F: PrimeField> {
    /// he coefficients corresponding to the `poly` in `SumcheckOracleRef`
    pub coeffs: DensePolynomial<F>,
    /// Degree of the polynomial input
    pub degree: usize,
    /// evaluation domain
    pub evaluation_domain: Radix2EvaluationDomain<F>,
    /// summation domain
    pub summation_domain: Radix2EvaluationDomain<F>,
}
```

The (RS-)IOP interface requires that verifier parameter is the subset of prover parameter, and let's show the prover parameter is valid. 

```rust
impl<F: PrimeField> ProverParam for SumcheckProverParameter<F> {
    type VerifierParameter = SumcheckVerifierParameter<F>;

    fn to_verifier_param(&self) -> Self::VerifierParameter {
        Self::VerifierParameter {
            degree: self.degree,
            evaluation_domain: self.evaluation_domain,
            summation_domain: self.summation_domain,
        }
    }
}
```

Additionally, verifier needs to have oracle access to oracle evaluations. In almost all cases, the oracle evaluations is already sent by prover previously in the same transcript, so univariate sumcheck verifier only needs an oracle reference. In code: 

```rust
#[derive(Clone, Debug, Copy)]
pub struct SumcheckOracleRef {
    poly: MsgRoundRef,
}

impl SumcheckOracleRef {
    pub fn new(poly: MsgRoundRef) -> Self {
        SumcheckOracleRef { poly }
    }
}
```

Again, oracle reference for prover is a superset of oracle reference for verifier, in code, we need to implement this trait to show this fact: 

```rust
impl ProverOracleRefs for SumcheckOracleRef {
    type VerifierOracleRefs = Self;

    fn to_verifier_oracle_refs(&self) -> Self::VerifierOracleRefs {
        self.clone()
    }
}
```

The protocol reads **claimed sum** in public input. It is sometimes tricky to determine what puts to verifier parameter and what puts to public input. `ark-bcs` implementation requires that public-input should not change the structure of transcript, including number of rounds, type of message at each round, or degree bound. Therefore, verifier do not have access to public input in `register_iop_structure` function, which simulates interaction with prover in commit phase using succinct proof. 

Also, one `MsgRoundRef` represents one round, and one round can have multiple oracles with same length sharing one merkle tree, sumcheck protocol also needs to know which oracle in that round is the oracle we want. In code, we need this as public input: 

```rust
#[derive(Clone)]
pub struct SumcheckPublicInput<F: PrimeField + Absorb> {
    claimed_sum: F,
    /// `SumcheckOracleRef` represents one round, which can contain multiple
    /// oracles. Which oracle do we want to look at?
    which: usize,
}
```

Ideally every implementation in a virtual struct that implements `IOPProver` and `IOPVerifier` trait. For `SimpleSumcheck`, we can define the following struct, and implement the two traits: 

```rust
pub struct SimpleSumcheck<F: PrimeField + Absorb> {
    _field: PhantomData<F>,
}

impl<F: PrimeField + Absorb> IOPProver<F> for SimpleSumcheck<F> {
    type ProverParameter = SumcheckProverParameter<F>;
    type RoundOracleRefs = SumcheckOracleRef;
    type PublicInput = SumcheckPublicInput<F>;
    type PrivateInput = ();

    fn prove<MT: Config<Leaf = [F]>, S: CryptographicSponge>(
        namespace: NameSpace,
        oracle_refs: &Self::RoundOracleRefs,
        public_input: &Self::PublicInput,
        _private_input: &Self::PrivateInput,
        transcript: &mut Transcript<MT, S, F>,
        prover_parameter: &Self::ProverParameter,
    ) -> Result<(), ark_bcs::Error>
    where
        MT::InnerDigest: Absorb,
    {todo!();}
}

impl<S: CryptographicSponge, F: PrimeField + Absorb> IOPVerifier<S, F> for SimpleSumcheck<F> {
    type VerifierOutput = bool;
    type VerifierParameter = SumcheckVerifierParameter<F>;
    type OracleRefs = SumcheckOracleRef;
    type PublicInput = SumcheckPublicInput<F>;

    fn register_iop_structure<MT: Config<Leaf = [F]>>(
        namespace: NameSpace,
        transcript: &mut SimulationTranscript<MT, S, F>,
        verifier_parameter: &Self::VerifierParameter,
    ) where
        MT::InnerDigest: Absorb,
    {
        todo!();
    }

    fn query_and_decide<O: RoundOracle<F>>(
        namespace: NameSpace,
        verifier_parameter: &Self::VerifierParameter,
        public_input: &Self::PublicInput,
        oracle_refs: &Self::OracleRefs, 
        random_oracle: &mut S,
        messages_in_commit_phase: &mut MessagesCollection<&mut O, VerifierMessage<F>>,
    ) -> Result<Self::VerifierOutput, Error> {
        todo!();
    }
}
```

Note that any protocol that does not need to access oracles from other protocols (i.e. `OracleRefs = ()`) can be automatically made into a succinct proof using BCS transform. For example, suppose we have protocol `A`, which uses subprotocol `B`. `A` does not uses any oracles outside its scope, but verifier of `B` needs to access one oracle sent in `A`. In this case, protocol `A` can be accepted by BCS transform, but protocol `B` cannot be transformed without defining a wrapper. For `SimpleSumcheck`, because it does need to access oracle outside its scope, `SimpleSumcheck` can only be used as a subprotocol and cannot directly be transformed to a succinct proof using BCS. 

Now let's build the main prover and verifier code! First, let's review what prover and verifier do in this protocol. Suppose the polynomial we takes some is `f(x)`, and let `H` be summation domain, `V_H(x)` be vanishing polynomial of the summation domain, `|H| `be size of summation domain. 

- Prover compute `h(x)` with `deg(h)=deg(f) - |H|`, and `p(x)` with `deg(p) < |H| - 1`, such that `f(x) = h(x)V_H(x) + (x*p(x) + claimed_sum/|H|)`. Prover send `h(x)` and `p(x)`, evaluated at evaluation domain. In this example, we simply use polynomial division algorithm defined in `ark-poly` but there'll be a lot of room to optimize if we want to use in production.
- Verifier sends **no** message in commit phase. 
- Verifier samples a random element `s` in evaluation domain, and queries `f(s)`, `h(s)`, `p(s)`, and computes `v_H(s)` locally. Then, it checks `f(s)` is equal to `h(s)*v_H(s) + (s*p(s) + claimed_sum/|H|)`
- LDT checks that `h(x)` enforces degree bound `deg(f) - |H|`; `p(x)` enforces degree bound `|H| - 2`. 

We will not go over proof of completeness and soundness for this protocol in this tutorial. Let's assume this protocol is correct. 

#### `prove`

Recall prover needs to write the prove function with the following signature: 

```rust
fn prove<MT: Config<Leaf = [F]>, S: CryptographicSponge>(
        namespace: NameSpace,
        oracle_refs: &Self::RoundOracleRefs,
        public_input: &Self::PublicInput,
        _private_input: &Self::PrivateInput,
        transcript: &mut Transcript<MT, S, F>,
        prover_parameter: &Self::ProverParameter,
    ) -> Result<(), ark_bcs::Error> {
```

As a first step, prover can perform a sanity check to make sure the coefficients provided in prover parameter is consistent with the referenced oracle evaluation. 

```rust
// sanity check: `coeffs` in prover parameter matches the referenced oracle
let actual_eval = &transcript
    .get_previously_sent_prover_round(oracle_refs.poly)
    .reed_solomon_codes()[public_input.which]
    .0;
let expected_eval = prover_parameter
    .coeffs
    .clone()
    .evaluate_over_domain(prover_parameter.evaluation_domain)
    .evals;
assert_eq!(&expected_eval, actual_eval);
```

Then, we calculate `h(x)` and `p(x)` using basic polynomial division algorithm. 

```rust
let (hx, gx) = prover_parameter
            .coeffs
            .divide_by_vanishing_poly(prover_parameter.summation_domain)
            .unwrap();
let claim_sum_over_size = DensePolynomial::from_coefficients_vec(vec![
    public_input.claimed_sum / F::from(prover_parameter.summation_domain.size as u128),
]);
let x = DensePolynomial::from_coefficients_vec(vec![F::zero(), F::one()]);
let (px, r) = DenseOrSparsePolynomial::from(gx + (-claim_sum_over_size))
.divide_with_q_and_r(&DenseOrSparsePolynomial::from(x))
.unwrap();
// remainder should be zero
assert!(r.is_zero());
// degree bound information
let hx_degree_bound =
prover_parameter.degree - prover_parameter.summation_domain.size as usize;
println!("hx: degree {}, bound {}", hx.degree(), hx_degree_bound);
let px_degree_bound = prover_parameter.summation_domain.size as usize - 2;
println!("px: degree {}, bound {}", px.degree(), px_degree_bound);
```

Then, prover can add those two polynomial as oracles going to be sent in current round. 

```rust
transcript.send_univariate_polynomial(hx_degree_bound, &hx)?;
transcript.send_univariate_polynomial(px_degree_bound, &px)?;
```

The backend will automatically turns the coefficients to evaluations on evaluation domain. Transcript will get the evaluation domain from LDT parameter passed in to BCS, and it's caller's responsibility to make sure evaluation domain in parameter is consistent with evaluation domain in LDT. The protocol will fail otherwise. 

Then don't forget to submit this round! We can add a `&'static str` in `iop_trace!` here to help us debugging. When backend found something strange, it will this string and line number of the `iop_trace!` macro for assistance. 

```rust
transcript.submit_prover_current_round(namespace, iop_trace!("sumcheck hx, px"))?;
}
```

#### `register_iop_structure`

Verifier needs to implement two functions, and one of them is `register_iop_structure`. `register_iop_structure` simulates interaction with prover, using the succinct proof. You can think this as the "`prove` code", but all prover message is already given for you.  All you need to do, is to replay the interaction and sample verifier message in commit phase as needed. For simple sumcheck example, let's start with function signature: 

```rust
fn register_iop_structure<MT: Config<Leaf = [F]>>(
        namespace: NameSpace,
        transcript: &mut SimulationTranscript<MT, S, F>,
        verifier_parameter: &Self::VerifierParameter,
    ) where
        MT::InnerDigest: Absorb,
    {
```

The verifier will tell the transcript that prover will send a message with a certain degree at this time, in code: 

```rust
let hx_degree_bound =
    verifier_parameter.degree - verifier_parameter.summation_domain.size as usize;
let px_degree_bound = verifier_parameter.summation_domain.size as usize - 1;
let expected_round_info = ProverRoundMessageInfo {
    num_message_oracles: 0,
    reed_solomon_code_degree_bound: vec![hx_degree_bound, px_degree_bound],
    oracle_length: verifier_parameter.evaluation_domain.size(),
    num_short_messages: 0,
    localization_parameter: 0, // ignored
};
```

Similar to `libiop`, In `ark-bcs` implementation, the backend will serialize a message into a number of cosets where each coset is encoded into a leaf. Doing so can greatly improve performance if verifier query is structured. `localization_parameter` is the size of each coset. Note that `localization_parameter` can be any arbitrary value here because we are sending an oracle with degree bound: in this case LDT will automatically manage the localization parameter. If current round contains only oracles without degree bound, then `localization_parameter` is meaningful. 

Again, remember to "submit" this round: 

```rust
transcript.receive_prover_current_round(
            namespace,
            expected_round_info,
            iop_trace!("sumcheck hx, px"),
        );
```

#### `query_and_decide`

In `query_and_decide` phase, verifier has oracle access to all prover messages sent in the transcript in commit phase, and access to all verifier messages (verifier messages from current protocol, and verifier messages from other protocols, using reference). Again, let's start with function signature

```rust
fn query_and_decide<O: RoundOracle<F>>(
    namespace: NameSpace,
    verifier_parameter: &Self::VerifierParameter,
    public_input: &Self::PublicInput,
    oracle_refs: &Self::OracleRefs, /* in parent `query_and_decide`, parent can fill out
                                     * this `oracle_refs` using the message in current
                                     * protocol */
    random_oracle: &mut S,
    messages_in_commit_phase: &mut MessagesCollection<&mut O, VerifierMessage<F>>,
) -> Result<Self::VerifierOutput, Error> {
```

*Technical Note*: When `query_and_decide` is called by BCS prover, round oracle is a wrapper to the entire message and records the verifier's query. When `query_and_decide` is called by BCS verifier, round oracle is a pointer to one message in a succinct proof with a mutable state storing read position, and will panic if verifier queries more positions than expected. 

First, let define some useful variable here: 

```rust
let evaluation_domain = verifier_parameter.evaluation_domain;
let summation_domain = verifier_parameter.summation_domain;
let claimed_sum = public_input.claimed_sum;
let evaluation_domain_log_size = evaluation_domain.log_size_of_group;
```

Then, let's use the built in sponge to sample a random element in evaluation domain: 

```rust
let query: usize = random_oracle
    .squeeze_bits(evaluation_domain_log_size as usize)
    .into_iter()
    .enumerate()
    .map(|(k, v)| (v as usize) << k)
    .sum::<usize>();
let query_point: F = evaluation_domain.element(query);
```

Next, let query `h(s)`and`p(s)`. Adding `iop_trace!` here can help us debugging in case something does wrong. 

```rust
let queried_points = messages_in_commit_phase
    .prover_message(namespace, 0)
    .query(&[query], iop_trace!("sumcheck query"))
    .pop()
    .unwrap();
let h_point = queried_points[0];
let p_point = queried_points[1];
```

We can calculate `V_H(s)` locally: 

```rust
let vh_point = summation_domain
    .vanishing_polynomial()
    .evaluate(&query_point);
```

Then, we can query `f(s)`. We can pass `oracle_refs.poly` to `messages_in_commit_phase` to get a reference to the oracle of `f`. 

```rust
let expected = messages_in_commit_phase.prover_message_using_ref(oracle_refs.poly).query(&[query], iop_trace!("oracle access to poly in sumcheck"))
    .remove(0)// there's only one query, so always zero
    .remove(public_input.which); // we want to get `which` oracle in this round
                                 // h(s) * v_h(s) + (s * p(s) + claimed_sum/summation_domain.size)
```

The result returned by query is a 2D vector. The first axis is the query index. Since we have only one query here, we can simply call`remove(0)` to get what we want. The second axis is the the oracle index in this round. Caller already specifies which oracle we are going to use in `public_input.which`. 

Finally, just check that  `f(s)` is equal to `h(s)*v_H(s) + (s*p(s) + claimed_sum/|H|)`

```rust
let actual: F = h_point * vh_point
            + (query_point * p_point + claimed_sum / F::from(summation_domain.size as u128));
return Ok(expected == actual);
}
```

**Well done!** You got it! You now learned how to construct a univariate sumcheck protocol using `ark-bcs`. Now let's start implementing out main protocol using sumcheck as a subroutine. 

*Notice that univariate sumcheck is, indeed, a public coin PCP. But our main toy example will become an IOP.*

#### Seems that low degree test is "missing"?

One important check to guarantee soundness of sumcheck is that verifier must ensure `h(x)` and `p(x)` are bounded by a certain degree, but from above it seems that we do not write code for LDT check. In fact, this is one highlighted feature for RS-IOP, where **LDT check is seamless and automatic**. As long as verifier specifies the degree bound of the polynomial she plans to receive at this round, all LDT check will be automatically performed by BCS algorithm. There's a built-in LDT implementation, which uses FRI protocol defined in [`ark-ldt`](https://github.com/arkworks-rs/ldt). User can also use their own LDT implementation, though at current version, implementing a custom LDT is a little bit tricky. In future versions, there might not be an LDT interface, and a LDT checker will implementation `IOPProver` and `IOPVerifier` trait instead. 

## Build Main Protocol

Now you learned how to write univariate sumcheck using `ark-bcs` backend.  In this section, we will guide you to write the main protocol using sumcheck as a black box. There are a few blanks that you need to fill in order to check your understanding. A working solution can be found in [`examples/sumcheck/main.rs`](./main.rs).

First, as usual, think about what goes to parameter, public input and private input.  Go back to [Example Protocol Spec](#example-protocol-spec) to review the example protocol. 

```rust
#[derive(Clone, Debug)]
pub struct Parameter<F: PrimeField + Absorb> {
    evaluation_domain: Radix2EvaluationDomain<F>,
    summation_domain: Radix2EvaluationDomain<F>
    degrees: (usize, usize)
}

impl<F: PrimeField + Absorb> ProverParam for Parameter<F> {
    // hint: is verifier parameter different from prover parameter?
    type VerifierParameter = _____;

    fn to_verifier_param(&self) -> Self::VerifierParameter {
        _________________________________
    }
}

pub struct PublicInput<F: PrimeField + Absorb> {
    sums: (F, F), // sum of `poly0` over summation domain, sum of `poly1` over summation domain
}

pub struct PrivateInput<F: PrimeField + Absorb>(
    DensePolynomial<F>, // `poly0` coefficients
    DensePolynomial<F>, // `poly1` coefficients
);
```

Before going to next part, check out solution to make sure those parameters are correct!

#### `prove`

As usual, let's start writing `SumcheckExample<F>::prove` .

```rust
pub struct SumcheckExample<F: PrimeField + Absorb> {
    _field: PhantomData<F>,
}

impl<F: PrimeField + Absorb> IOPProver<F> for SumcheckExample<F> {
    type ProverParameter = Parameter<F>;
    type RoundOracleRefs = ();
    type PublicInput = PublicInput<F>;
    type PrivateInput = PrivateInput<F>;

    fn prove<MT: Config<Leaf = [F]>, S: CryptographicSponge>(
        namespace: NameSpace,
        _oracle_refs: &Self::RoundOracleRefs,
        public_input: &Self::PublicInput,
        private_input: &Self::PrivateInput,
        transcript: &mut Transcript<MT, S, F>,
        prover_parameter: &Self::ProverParameter,
    ) -> Result<(), Error>
    where
        MT::InnerDigest: Absorb,
    {
```

Write code to let verifier send two random field elements `r0` and `r1`. 

```rust
// receive two random combination
let random_coeffs = transcript
	.squeeze_verifier_field_elements(&[FieldElementSize::Full, FieldElementSize::Full]);
transcript.submit_verifier_current_round(namespace, iop_trace!("Verifier Random Coefficients"));
```

Prover will then multiply `poly0` by `r0`, and `poly1` by `r1`. The asserted sum also needs to be multiplied correspondingly. 

```rust
// multiply each polynomial in private input by the coefficient
let poly0 = DensePolynomial::from_coefficients_vec(
    private_input
        .0
        .coeffs
        .iter()
        .map(|coeff| *coeff * &random_coeffs[0])
        .into_iter()
        .collect::<Vec<_>>(),
);
let asserted_sum0 = public_input.sums.0 * random_coeffs[0];
let poly1 = DensePolynomial::from_coefficients_vec(
    private_input
        .1
        .coeffs
        .iter()
        .map(|coeff| *coeff * &random_coeffs[1])
        .into_iter()
        .collect::<Vec<_>>(),
);
let asserted_sum1 = public_input.sums.1 * random_coeffs[1];
```

 Write code to let send evaluations of `poly0`, `poly1` over evaluation domain, and store a reference to the round submitted to `round_ref`.  

```rust
// Hint: check out univariate sumcheck example 
transcript.______________________________________;
transcript.______________________________________;
// always remember to submit this round
let round_ref: MsgRoundRef = 
    transcript.submit_prover_current_round(namespace, iop_trace!("two polynomials for sumcheck"))?;
```

Next invoke univariate sumcheck to prove sum of `r0*poly0`. First, we need to give subprotocol a new namespace, so that the subprotocol interactions will not affect current namespace. We can do so by

```rust
let ns0 = transcript.new_namespace(namespace, iop_trace!("first sumcheck protocol"));
```

Then, we can just call `prove`.

```rust
SimpleSumcheck::prove(
    ns0, // namespace
    &SumcheckOracleRef::new(round_ref), //oracle_refs: reference to `poly0`
    &SumcheckPublicInput::new(asserted_sum0, 0 /*which oracle in round*/),
    &(), // pricate input is not needed for simple univariate sumcheck 
    transcript,
    &SumcheckProverParameter {
        coeffs: poly0,
        summation_domain: prover_parameter.summation_domain,
        evaluation_domain: prover_parameter.evaluation_domain,
        degree: prover_parameter.degrees.0,
    },
)?;
```

Finally, invoke univariate sumcheck to prover sum of `r1*poly1`. It should be quite similar to last code block. 

```rust
transcript.new_namespace(ns1.clone(), iop_trace!("second sumcheck protocol 1"));
SimpleSumcheck::prove(
    ns1,
    &SumcheckOracleRef::new(round_ref),
    &SumcheckPublicInput::new(asserted_sum1, 1),
    &(),
    transcript,
    &SumcheckProverParameter {
        coeffs: poly1,
        summation_domain: prover_parameter.summation_domain,
        evaluation_domain: prover_parameter.evaluation_domain,
        degree: prover_parameter.degrees.1,
    },
)?;
```

Finally, we can end the function by returning

```rust
Ok(())
```



#### `register_iop_structure`

Writing `register_iop_structure` is easy. Just copy the `prove` function, and keeps only the interaction with transcript. All prover computation can be removed. 

```rust
impl<S: CryptographicSponge, F: PrimeField + Absorb> IOPVerifier<S, F> for SumcheckExample<F> {
    type VerifierOutput = bool;
    type VerifierParameter = Parameter<F>;
    type OracleRefs = ();
    type PublicInput = PublicInput<F>;

    fn register_iop_structure<MT: Config<Leaf = [F]>>(
        namespace: NameSpace,
        transcript: &mut SimulationTranscript<MT, S, F>,
        verifier_parameter: &Self::VerifierParameter,
    ) where
        MT::InnerDigest: Absorb,
    {
        // verifier send two random combination
        transcript.___________________________________________;
        transcript.___________________________________________;
        
        // receive two prover oracles in one round.
        transcript.receive_prover_current_round(
            namespace,
            ProverRoundMessageInfo::new(
                vec![verifier_parameter.degrees.0, verifier_parameter.degrees.1],
                0,
                0,
                verifier_parameter.evaluation_domain.size(),
                0,
            ),
            iop_trace!("two polynomials for sumcheck"),
        );

        // invoke sumcheck protocol on first protocol
        let ns0 = transcript.new_namespace(namespace, iop_trace!("first sumcheck protocol"));

        SimpleSumcheck::register_iop_structure(
            &ns0,
            transcript,
            &SumcheckVerifierParameter {
                degree: verifier_parameter.degrees.0,
                evaluation_domain: verifier_parameter.evaluation_domain,
                summation_domain: verifier_parameter.summation_domain,
            },
        );

        // invoke sumcheck protocol on second protocol
        let ns1 = transcript.new_namespace(namespace, iop_trace!("first sumcheck protocol"));

        SimpleSumcheck::register_iop_structure(
            &ns1,
            transcript,
            &SumcheckVerifierParameter {
                degree: verifier_parameter.degrees.1,
                evaluation_domain: verifier_parameter.evaluation_domain,
                summation_domain: verifier_parameter.summation_domain,
            },
        );
    }
```

#### `query_and_decide`

Finally, let's write verifier's `query_and_decide` phase. 

```rust
fn query_and_decide<O: RoundOracle<F>>(
    namespace: NameSpace,
    verifier_parameter: &Self::VerifierParameter,
    public_input: &Self::PublicInput,
    _oracle_refs: &Self::OracleRefs,
    sponge: &mut S,
    messages_in_commit_phase: &mut MessagesCollection<&mut O, VerifierMessage<F>>,
) -> Result<Self::VerifierOutput, Error> {
```

First, let's get a reference to the oracle that prover just sent in this protocol. 

```rust
let oracle_refs_sumcheck =
    SumcheckOracleRef::new(*messages_in_commit_phase.prover_message_as_ref(namespace, 0));
```

Now get the random coefficients the verifier sampled in commit phase, and compute the asserted sums of `r0*poly[0]`, `r1*poly[1]` using those coefficients. 

```rust
let random_coeffs = messages_in_commit_phase.verifier_message(namespace, 0)[0]
    .clone()
    .try_into_field_elements()
    .expect("invalid verifier message type");
let asserted_sums = (
    public_input.sums.0 * random_coeffs[0],
    public_input.sums.1 * random_coeffs[1],
);
```

Finally, let's invoke sumcheck subprotocols! In `query`, we will instead use `messages_in_commit_phase` struct to get subspace. `messages_in_commit_phase.get_subprotocol_namespace(namespace, 0)` will return the first subspace created during `register_iop_structure`. Similarly, `messages_in_commit_phase.get_subprotocol_namespace(namespace, 1)` will return the second subspace created during `register_iop_structure`. 

```rust
// invoke first sumcheck protocol
let mut result = SimpleSumcheck::query_and_decide(
    messages_in_commit_phase.get_subprotocol_namespace(namespace, 0),
    &SumcheckVerifierParameter {
        degree: verifier_parameter.degrees.0,
        evaluation_domain: verifier_parameter.evaluation_domain,
        summation_domain: verifier_parameter.summation_domain,
    },
    &SumcheckPublicInput::new(asserted_sums.0, 0),
    &oracle_refs_sumcheck,
    sponge,
    messages_in_commit_phase,
)?;

// invoke second sumcheck protocol
result &= SimpleSumcheck::query_and_decide(
    messages_in_commit_phase.get_subprotocol_namespace(namespace, 1),
    &SumcheckVerifierParameter {
        degree: verifier_parameter.degrees.1,
        evaluation_domain: verifier_parameter.evaluation_domain,
        summation_domain: verifier_parameter.summation_domain,
    },
    &SumcheckPublicInput::new(asserted_sums.1, 1),
    &oracle_refs_sumcheck,
    sponge,
    messages_in_commit_phase,
)?;

Ok(result)
```

**Congratulations!** You now built your first RS-IOP using `ark-bcs` library. Next, we will show you how to transform this interactive protocol to a succinct non-interactive proof using BCS transform. 

## Put it together: Proof Generation

Now we have written an interactive protocol for univariate sumcheck. By using `ark-bcs` backend, it will be quite easy to convert it to a succinct proof. 

First let's make up a context. Suppose we have `poly1` and `poly2` from somewhere else. For now, to make this tutorial self-contained, the polynomial is just some random garbage. 

```rust
fn main() {
    let mut rng = test_rng();
    let degrees = (155, 197);
    let poly0 = DensePolynomial::<Fr>::rand(degrees.0, &mut rng);
    let poly1 = DensePolynomial::<Fr>::rand(degrees.1, &mut rng);
```

Then, we define its summation domain and evaluation domain. 

```rust
let summation_domain = Radix2EvaluationDomain::new(64).unwrap();
let evaluation_domain = Radix2EvaluationDomain::new(512).unwrap();
```

Then, suppose we also have the claimed sum ready. 

```rust
let claimed_sum1 = Radix2CosetDomain::new(summation_domain.clone(), Fr::one())
    .evaluate(&poly0)
    .into_iter()
    .sum::<Fr>();
let claimed_sum2 = Radix2CosetDomain::new(summation_domain.clone(), Fr::one())
    .evaluate(&poly1)
    .into_iter()
    .sum::<Fr>();
```

We will use built in LDT implementation, we runs one FRI on a random linear combination of all polynomial with degree bound. For example, we have `f_1(x)` and `f_2(x)`, and FRI has degree bound `D`. This protocol first sample randomness `r1` and `r2` and runs FRI on `r1*f_1(x)*x^{D-deg(f_1)} + r2*f_2(x)*x^{D-deg(f_2)}`. FRI's degree bound should be larger than the max degree of all polynomials, and should be a power of two. 

We will use FRI with degree bound `256`. At first bound, we shrink the domain by `1/2^1`, second round `1/2^3`, third round `1/2`, so the localization parameter will be `[1,3,1]`. 

```rust
let fri_parameters = FRIParameters::new(
    256,
    vec![1, 3, 1],
    Radix2CosetDomain::new(evaluation_domain, Fr::one()),
);
```

We will run the LDT using 3 FRI queries: 

```rust
let ldt_parameter = LinearCombinationLDTParameters {
    fri_parameters,
    num_queries: 3,
};
```

Then, let's define all things needed for prover: 

```rust
let sponge = PoseidonSponge::new(&poseidon_parameters());
let mt_hash_parameters = MTHashParameters::<FieldMTConfig> {
    leaf_hash_param: poseidon_parameters(),
    inner_hash_param: poseidon_parameters(),
};

let vp = PublicInput {
    sums: (claimed_sum1, claimed_sum2),
};
let wp = PrivateInput(poly0, poly1);
let prover_param = Parameter {
    degrees,
    summation_domain,
    evaluation_domain,
};
```

Generating proof is just "one line":

```rust
let proof = BCSProof::generate::<
    SumcheckExample<Fr>, // Need to implement IOPProver
    SumcheckExample<Fr>, // Need to implement IOPVerifier Trait
    LinearCombinationLDT<Fr>, // LDT implementation
    _, // Sponge type
>(
    sponge,
    &vp,
    &wp,
    &prover_param,
    &ldt_parameter,
    mt_hash_parameters.clone(),
)
.expect("fail to generate proof");
```

Proof is serializable regardless of implementation of prover and verifier. We can also check the proof size in bytes: 

```rust
println!("Proof Size: {} bytes", proof.serialized_size());
```

Finally let's verify if the proof is correct! As we already generated some prover parameters, let's do some cheating by deriving verifier parameter from prover parameter. In real circumstance, just making them again using same code. 

```rust
let sponge = PoseidonSponge::new(&poseidon_parameters());
let verifier_param = prover_param.to_verifier_param();
```

Verifying a proof is also "one line": 

```rust
let result = BCSVerifier::verify::<SumcheckExample<Fr>, LinearCombinationLDT<Fr>, _>(
    sponge,
    &proof,
    &vp,
    &verifier_param,
    &ldt_parameter,
    mt_hash_parameters.clone(),
)
.expect("fail to verify");
assert!(result);
```

And that's it! You've learned how to write RS-IOP using this library! 

#### Under the hood

You might be curious on what exact does `BCSProver::prove` do. To see what goes under the hood, go to `ark-bcs` project path, and run the example with `print-trace` feature enabled: 

```shell
cargo run --package ark-bcs --example sumcheck --features test_utils
```

At first few rounds, we can see `SumcheckExample::prove` is being run: 

```log
Oct 02 21:19:58.208  INFO submit_prover_current_round{namespace=[BCS Proof Generation: Commit Phase] at src\bcs\prover.rs:89:29}: ark_bcs::bcs::transcript: [two polynomials for sumcheck] at examples\sumcheck\main.rs:132:53
Oct 02 21:19:58.518  INFO submit_prover_current_round{namespace=[first sumcheck protocol] at examples\sumcheck\main.rs:135:55}: ark_bcs::bcs::transcript: [sumcheck hx, px] at examples\sumcheck\simple_sumcheck.rs:144:59
Oct 02 21:19:58.830  INFO submit_prover_current_round{namespace=[second sumcheck protocol 1] at examples\sumcheck\main.rs:151:55}: ark_bcs::bcs::transcript: 
[sumcheck hx, px] at examples\sumcheck\simple_sumcheck.rs:144:59
```

Then BCS, BCS will run LDT commit phase, and attach message oracle to a separate LDT transcript. 

```
Oct 02 21:19:59.134  INFO submit_verifier_current_round{namespace=[LDT Prove] at src\ldt\rl_ldt.rs:60:41}: ark_bcs::bcs::transcript: [ldt random coefficeints] at src\ldt\rl_ldt.rs:71:55
Oct 02 21:19:59.154  INFO submit_verifier_current_round{namespace=[LDT Prove] at src\ldt\rl_ldt.rs:60:41}: ark_bcs::bcs::transcript: [ldt alpha] at src\ldt\rl_ldt.rs:118:67
Oct 02 21:19:59.156  INFO submit_prover_current_round{namespace=[LDT Prove] at src\ldt\rl_ldt.rs:60:41}: ark_bcs::bcs::transcript: [ldt fri oracle] at src\ldt\rl_ldt.rs:133:65
Oct 02 21:19:59.219  INFO submit_verifier_current_round{namespace=[LDT Prove] at src\ldt\rl_ldt.rs:60:41}: ark_bcs::bcs::transcript: [ldt alpha] at src\ldt\rl_ldt.rs:118:67
Oct 02 21:19:59.221  INFO submit_prover_current_round{namespace=[LDT Prove] at src\ldt\rl_ldt.rs:60:41}: ark_bcs::bcs::transcript: [ldt fri oracle] at src\ldt\rl_ldt.rs:133:65
Oct 02 21:19:59.234  INFO submit_verifier_current_round{namespace=[LDT Prove] at src\ldt\rl_ldt.rs:60:41}: ark_bcs::bcs::transcript: [ldt final alpha] at src\ldt\rl_ldt.rs:142:65
Oct 02 21:19:59.236  INFO submit_prover_current_round{namespace=[LDT Prove] at src\ldt\rl_ldt.rs:60:41}: ark_bcs::bcs::transcript: [ldt final poly coefficients] at src\ldt\rl_ldt.rs:168:53
```

Then, BCS will run LDT verifier's `query_and_decision` phase, to record query responses.

```
Oct 04 20:29:33.547  INFO query_coset{coset_index=[49]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63Oct 04 20:29:33.547  INFO query_coset{coset_index=[49]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63Oct 04 20:29:33.548  INFO query_coset{coset_index=[49]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63Oct 04 20:29:33.548  INFO query_coset{coset_index=[17]}: ark_bcs::iop::message: [rl_ldt query fri message] at src\ldt\rl_ldt.rs:321:59
Oct 04 20:29:33.548  INFO query_coset{coset_index=[1]}: ark_bcs::iop::message: [rl_ldt query fri message] at src\ldt\rl_ldt.rs:321:59
Oct 04 20:29:33.550  INFO query_coset{coset_index=[243]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63
Oct 04 20:29:33.551  INFO query_coset{coset_index=[243]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63
Oct 04 20:29:33.551  INFO query_coset{coset_index=[243]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63
Oct 04 20:29:33.552  INFO query_coset{coset_index=[19]}: ark_bcs::iop::message: [rl_ldt query fri message] at src\ldt\rl_ldt.rs:321:59
Oct 04 20:29:33.552  INFO query_coset{coset_index=[3]}: ark_bcs::iop::message: [rl_ldt query fri message] at src\ldt\rl_ldt.rs:321:59
Oct 04 20:29:33.555  INFO query_coset{coset_index=[241]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63
Oct 04 20:29:33.556  INFO query_coset{coset_index=[241]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63
Oct 04 20:29:33.556  INFO query_coset{coset_index=[241]}: ark_bcs::iop::message: [rl_ldt query codewords] at src\ldt\rl_ldt.rs:283:63
Oct 04 20:29:33.557  INFO query_coset{coset_index=[17]}: ark_bcs::iop::message: [rl_ldt query fri message] at src\ldt\rl_ldt.rs:321:59
Oct 04 20:29:33.557  INFO query_coset{coset_index=[1]}: ark_bcs::iop::message: [rl_ldt query fri message] at src\ldt\rl_ldt.rs:321:59
```

Finally, BCS will run `SumcheckExample::query_and_decide` to run the verifier and record query responses. 

```
Oct 04 20:29:33.558  INFO query{position=[429]}: ark_bcs::iop::message: [sumcheck query] at examples\sumcheck\simple_sumcheck.rs:207:30
Oct 04 20:29:33.559  INFO query{position=[429]}: ark_bcs::iop::message: [oracle access to poly in sumcheck] at examples\sumcheck\simple_sumcheck.rs:215:108
Oct 04 20:29:33.559  INFO query{position=[83]}: ark_bcs::iop::message: [sumcheck query] at examples\sumcheck\simple_sumcheck.rs:207:30
Oct 04 20:29:33.560  INFO query{position=[83]}: ark_bcs::iop::message: [oracle access to poly in sumcheck] at examples\sumcheck\simple_sumcheck.rs:215:108
```

You can check what `BCSVerifier::verify` did in a similar way. 

## Bonus: Write circuits for the Verifier

`ark-bcs` also provides framework to write circuits for RS-IOP verifier. The circuits generated will be R1CS constraints, and can be fed to any SNARKs that implements `Snark` trait defined in [`ark-snark`](https://github.com/arkworks-rs/snark/tree/master/snark). Let's now write constraints of verifier of the example sumcheck protocol!

First, let's define the variable for sumcheck public input: 

```rust
pub struct SumcheckPublicInputVar<CF: PrimeField + Absorb> {
    claimed_sum: FpVar<CF>,
    which: usize,
}

impl<CF: PrimeField + Absorb> SumcheckPublicInputVar<CF> {
    pub fn new(claimed_sum: FpVar<CF>, which: usize) -> Self {
        SumcheckPublicInputVar { claimed_sum, which }
    }
}
```

Next, we will implement `IOPVerifierWithGadget` trait for `SimpleSumcheck`. The output of verifier circuit will be a `Boolean` variable.

```rust
impl<CF: PrimeField + Absorb, S: SpongeWithGadget<CF>> IOPVerifierWithGadget<S, CF>
    for SimpleSumcheck<CF>
{
    type VerifierOutputVar = Boolean<CF>;
    type PublicInputVar = SumcheckPublicInputVar<CF>;
```

Then, we will implement those two functions. 

```rust
fn register_iop_structure_var<MT: Config, MTG: ConfigGadget<MT, CF, Leaf = [FpVar<CF>]>>(
    namespace: NameSpace,
    transcript: &mut SimulationTranscriptVar<CF, MT, MTG, S>,
    verifier_parameter: &Self::VerifierParameter,
) -> Result<(), SynthesisError>
where
    MT::InnerDigest: Absorb,
    MTG::InnerDigest: AbsorbGadget<CF>;

fn query_and_decide_var(
    _cs: ConstraintSystemRef<CF>,
    namespace: NameSpace,
    verifier_parameter: &Self::VerifierParameter,
    public_input: &Self::PublicInputVar,
    oracle_refs: &Self::OracleRefs,
    sponge: &mut S::Var,
    messages_in_commit_phase: &mut MessagesCollection<
        &mut SuccinctRoundOracleVarView<CF>,
        VerifierMessageVar<CF>,
    >,
) -> Result<Self::VerifierOutputVar, SynthesisError>
```

The code should be quite similar to native code. Try to implement them yourself! 

*Hint: In native code, when we want to sample an integer (e.g`u32`), we first call `sponge.squeeze_bits`, and convert those bits to an integer in little endian order. In constraints, when calculate `query_point`, we use `pow_le` that directly takes the squeezed bits, interpreted in little endian order.*

You can check our implementation at https://github.com/arkworks-rs/bcs/blob/main/examples/sumcheck/simple_sumcheck.rs. 

Writing main code for the protocol is also quite similar to the native version, and you can out implementation at https://github.com/arkworks-rs/bcs/blob/work/examples/sumcheck/constraints.rs. 

Now let's write a constraints synthesizer. We will need those info to generate our circuit: 

```rust
struct SumcheckExampleVerification {
    // Constants embedded into the circuit: some parameters, for example
    param: Parameter<Fr>,
    poseidon_param: PoseidonParameters<Fr>, /* for simplicity, same poseidon parameter is used
                                             * for both merkle tree, and sponge */
    ldt_param: LinearCombinationLDTParameters<Fr>,

    // public input is the public input known by the verifier
    vp: PublicInput<Fr>,

    // private witness: we want to show we know the proof for the sumcheck example
    proof: BCSProof<FieldMTConfig, Fr>,
}
```

Additionally, let's define the merkle tree config used in our circuit. For now we need to have two separate struct (one for native code, and one for constraints). The code will be simpler in the future (see https://github.com/arkworks-rs/crypto-primitives/pull/68). 

```rust
pub(crate) struct FieldMTConfig;
impl Config for FieldMTConfig {
    type Leaf = [Fr];
    type LeafDigest = Fr;
    type LeafInnerDigestConverter = IdentityDigestConverter<Fr>;
    type InnerDigest = Fr;
    type LeafHash = poseidon::CRH<Fr>;
    type TwoToOneHash = poseidon::TwoToOneCRH<Fr>;
}

pub(crate) struct FieldMTConfigGadget;
impl ConfigGadget<FieldMTConfig, Fr> for FieldMTConfigGadget {
    type Leaf = [FpVar<Fr>];
    type LeafDigest = FpVar<Fr>;
    type LeafInnerConverter = IdentityDigestConverter<FpVar<Fr>>;
    type InnerDigest = FpVar<Fr>;
    type LeafHash = poseidon::constraints::CRHGadget<Fr>;
    type TwoToOneHash = poseidon::constraints::TwoToOneCRHGadget<Fr>;
}
```

Next, let's generate the constraints, first we allocate some parameters as constant. 

```rust
impl ConstraintSynthesizer<Fr> for SumcheckExampleVerification {
    fn generate_constraints(self, cs: ConstraintSystemRef<Fr>) -> Result<(), SynthesisError> {
        // allocate some constants for parameters
        let poseidon_param_var = poseidon::constraints::CRHParametersVar::new_constant(
            cs.clone(),
            &self.poseidon_param,
        )?;
        let mt_hash_param_var = MTHashParametersVar::<Fr, FieldMTConfig, FieldMTConfigGadget> {
            leaf_params: poseidon_param_var.clone(),
            inner_params: poseidon_param_var.clone(),
        };
```

Then, we allocate public inputs, which is the public inputs variable for the main protocol. 

```rust
        // first, allocate public inputs
        let vp_var = PublicInputVar::new_input(ark_relations::ns!(cs, "public input"), || {
            Ok(self.vp.clone())
        })?;
```

Then, we allocate proof as a private input. 

```rust
        // allocate proof
        let proof_var =
            BCSProofVar::new_witness(ark_relations::ns!(cs, "proof"), || Ok(self.proof.clone()))?;
```

Finally, let's constrain the `proof_var` using the verifier gadget we just wrote.

```rust
        // write constraints to enforce the proof correctness
        let result = BCSVerifierGadget::verify::<
            SumcheckExample<Fr>,
            LinearCombinationLDT<Fr>,
            PoseidonSponge<Fr>,
        >(
            ark_relations::ns!(cs, "proof verification").cs(),
            PoseidonSpongeVar::new(cs.clone(), &self.poseidon_param),
            &proof_var,
            &vp_var,
            &self.param,
            &self.ldt_param,
            &mt_hash_param_var,
        )?;

        result.enforce_equal(&Boolean::TRUE)?;
        Ok(())
    }
}
```

And that's it! 

## Bonus: Debug Your IOP Code using `iop_trace!`

You may recall that anytime we send message using the transcript, or query messages from the oracles, we use a macro called `iop_trace! `. It basically records the position where this macro is called, and when something goes wrong, it can output which one goes wrong. 

One particularly useful application is to check if `register_iop_structure` is consistent with prover code. Now, let's first write a test called `test_register`. Same as `main` code, we generate all necessary parameters and inputs for prover and verifier. 

```rust
#[test]
fn test_register() {
    let mut rng = test_rng();
    let degrees = (155, 197);
    let poly0 = DensePolynomial::<Fr>::rand(degrees.0, &mut rng);
    let poly1 = DensePolynomial::<Fr>::rand(degrees.1, &mut rng);
    let summation_domain = Radix2EvaluationDomain::new(64).unwrap();
    let evaluation_domain = Radix2EvaluationDomain::new(512).unwrap();
    let fri_parameters = FRIParameters::new(
        256,
        vec![1, 3, 1],
        Radix2CosetDomain::new(evaluation_domain, Fr::one()),
    );
    let ldt_parameter = LinearCombinationLDTParameters {
        fri_parameters,
        num_queries: 3,
    };
    let claimed_sum1 = Radix2CosetDomain::new(summation_domain.clone(), Fr::one())
        .evaluate(&poly0)
        .into_iter()
        .sum::<Fr>();
    let claimed_sum2 = Radix2CosetDomain::new(summation_domain.clone(), Fr::one())
        .evaluate(&poly1)
        .into_iter()
        .sum::<Fr>();

    let sponge = PoseidonSponge::new(&poseidon_parameters());
    let mt_hash_parameters = MTHashParameters::<FieldMTConfig> {
        leaf_hash_param: poseidon_parameters(),
        inner_hash_param: poseidon_parameters(),
    };

    let vp = PublicInput {
        sums: (claimed_sum1, claimed_sum2),
    };
    let wp = PrivateInput(poly0, poly1);
    let prover_param = Parameter {
        degrees,
        summation_domain,
        evaluation_domain,
    };
```

Then, we can invoke call `ark_bcs::bcs::transcript::test_utils::check_commit_phase_correctness` helper function, which will panic if `register_iop_structure` is inconsistent with the prover code. 

*Note: make sure `test_utils` feature is on.*

```rust
check_commit_phase_correctness::<
        _,
        _,
        _,
        SumcheckExample<_>,
        SumcheckExample<_>,
        LinearCombinationLDT<_>,
    >(
        sponge,
        &vp,
        &wp,
        &prover_param,
        &ldt_parameter,
        mt_hash_parameters.clone(),
    );
}
```

The test is also provided in the `examples/sumcheck ` folder of the repo. When you run this test using the following command, you should find the test passes when running this command:

```shell
cargo test --package ark-bcs --example sumcheck --features r1cs --features test_utils -- test_register --exact
```

Now, let's manually introduce a bug in `register_iop_structure`. Suppose the verifier tries to send a wrong message. Say: 

```rust
// verifier should sample two random combination, but sampled three!
        transcript
            .squeeze_verifier_field_elements(&[FieldElementSize::Full, FieldElementSize::Full, FieldElementSize::Full]);
```

 Now let's run the test again! The test should fail, showing the following message: 

```log
thread 'test_register' panicked at 'assertion failed: `(left == right)`
  left: `[FieldElements([Fp256(BigInteger256([15868405394323405676, 15996445718776549797, 6686138471952702797, 6934092815371758962])), Fp256(BigInteger256([5499267208319786770, 1299149589602166132, 2231165342563635412, 5630332791626216984]))])]`,
 right: `[FieldElements([Fp256(BigInteger256([15868405394323405676, 15996445718776549797, 6686138471952702797, 6934092815371758962])), Fp256(BigInteger256([5499267208319786770, 1299149589602166132, 2231165342563635412, 5630332791626216984])), Fp256(BigInteger256([15777789622597396422, 11153390519922978925, 10843330044712850024, 3170697828371625695]))])]`: Inconsistent verifier round #0.
Prover transcript message trace: [Verifier Random Coefficients] at examples\sumcheck\main.rs:103:55
Verifier transcript message trace: [Verifier Random Coefficients] at examples\sumcheck\main.rs:190:55
Prover transcript defines this namespace as [Check commit phase correctness] at src\bcs\transcript.rs:1039:17
Verifier defines this namespace as [check commit phase correctness] at src\bcs\transcript.rs:1062:13
```

From this message, we can learn that at least one location of `examples\sumcheck\main.rs:103:55`, `examples\sumcheck\main.rs:190:55` goes wrong, which can greatly save us from detective works!

## Further Reading: Multi-round example

There's an alternative implementation of the example protocol, which contains multiple prover and verifier rounds in the root namespace. The specification of this alternative protocol is like this: 

| Prover                                            | Verifier                                 |
| ------------------------------------------------- | ---------------------------------------- |
|                                                   | Send random field element `r0`           |
| Send evaluations`r0*poly0`over evaluation domain. |                                          |
| Invoke univariate sumcheck on `r0*poly0`          | Invoke univariate sumcheck on `r0*poly0` |
|                                                   | Send random field element `r1`           |
| Send evaluations`r1*poly1`over evaluation domain. |                                          |
| Invoke univariate sumcheck on `r1*poly1`          | Invoke univariate sumcheck on `r1*poly1` |

Note that if original protocol where we sent `r0*poly0`, `r1*poly1` in one round, each query for those two polynomials share one authentication path, which can reduces the proof size. The proof size for multi-round example `~1KB` larger than original example. 

Code can be found in https://github.com/arkworks-rs/bcs/blob/main/examples/sumcheck/multiround_example.rs. If should look very similar to the protocol you just learned. 
