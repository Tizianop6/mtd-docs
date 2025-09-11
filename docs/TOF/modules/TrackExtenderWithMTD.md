# TrackExtenderWithMTD

!!! abstract
    This producer module (`EDProducer`) adds time and related information to the event, including:

    - time of arrival at MTD (\(t_\mathrm{MTD}\)) with uncertainty;
    - time of flight (henceforth TOF) under 3 different mass hypotheses (\(\pi\), \(K\), \(p\)) with uncertainty;
    - length of travelled path from primary vertex to MTD;
    - information on track matching to BTL, ETL hits (spatial and time match \(\chi^2\));
    - momentum, \(\beta\) and \(t_0\) of each track at innermost point.

## Class definition
Since `TrackExtenderWithMTD` class is designed to work with possibly different types of track collections, a template class `TrackExtenderwithMTDT` is defined as:
``` cpp
template <class TrackCollection>
class TrackExtenderWithMTDT : public edm::stream::EDProducer<>;
```

Although the collection can be changed, it is now meant to work with `reco::TrackCollection`:

``` cpp
typedef TrackExtenderWithMTDT<reco::TrackCollection> TrackExtenderWithMTD;
```

## Code flow

The main entrypoint is the `produce()` method. Although each called method is described in full detail the [Methods](#methods) section below, an overall summary is here provided. For each track, `produce()`:

1. first matches BTL and ETL hits to the track using the [`tryBTLLayers()`](#trybtllayers), [`tryETLLayers()`](#tryetllayers) methods. If matched, trajectory and track hit list is updated to also include MTD hits. The association with hits is what determines \(t_\mathrm{MTD}\) for the track. It should be noted that, in order for an approximate evaluation of time compatibility between the track and MTD hits, the [`trackPathLength()`](#trackpathlength) method is first invoked here, although its output is not used at this point for the actual TOF calculation.
2. By using the [`buildTrack()`](#buildtrack) method, the updated track is built, also calculating the path length between vertex and MTD, the associated TOF under different mass hypotheses and kinematic quantities obtained in the calculation. It is at this step that track back-propagation to vertex is performed, evaluating the track momentum at each tracker hit: all this information is contained in a [`TrackSegments`](#tracksegments) object, built by the [`trackPathLength()`](#trackpathlength) method. This object is then fed to the [`computeTrackTofPidInfo()`](#computetracktofpidinfo) method for TOF claculation, which saves all info to a [`TrackTofPidInfo`](#tracktofpidinfo) object.
Each rebuilt track is stored in a track collection that will be saved to the event in the final step.
3. By using the [`buildTrackExtra()`](#buildtrackextra) method, track extra information, most notably including its hit collection, is populated and associated to the track re-built in the previous step.
4. The rebuilt tracks, their associated track extras and associated reco hits are saved back into the `Event` (respectively as `TrackCollection`, `TrackExtraCollection`, `TrackingRecHit`). All previously evaluted time information is instead stored as a `ValueMap` and saved in the event through the [`fillValueMap()`](#fillvaluemap) method. 

The overall call flow is hence as follows:
``` bash
produce()
├── tryBTLLayers()
├── tryETLLayers()
├── trackPathLength()#(1)!
├── buildTrack()
│   ├── trackPathLength()
│   └── computeTrackTofPidInfo()
├── buildTrackExtra()
└── fillValueMap()
```

1.  called, but not used for actual TOF evaluation (second call is)

## Methods

### `tryBTLLayers()`

---

### `tryETLLayers()`

---

### `buildTrack()`

``` cpp title="Prototype"
template <class TrackCollection>
reco::Track TrackExtenderWithMTDT<TrackCollection>::buildTrack( //(1)!
    const reco::TrackRef& orig,
    const Trajectory& traj,
    const Trajectory& trajWithMtd,
    const reco::BeamSpot& bs,
    const MagneticField* field,
    const Propagator* thePropagator,
    bool hasMTD,
    float& pathLengthOut,
    float& tmtdOut,
    float& sigmatmtdOut,
    float& tofpi,
    float& tofk,
    float& tofp,
    float& sigmatofpi,
    float& sigmatofk,
    float& sigmatofp
    ) const;
```

1.  buildTrack is a template method (to which the `#!cpp template <class TrackCollection>` heading refers to) of a template class (hence `#!cpp TrackExtenderWithMTDT<TrackCollection>`) returning a `reco::Track` value

<!-- 
| Method      | Description                          |
| ----------- | ------------------------------------ |
| `float `    | :material-check:     Fetch resource  |
| `PUT`       | :material-check-all: Update resource |
| `DELETE`    | :material-close:     Delete resource |
-->

#### Steps

1. **Calculate path length** by invoking [`trackPathLength()`](#trackpathlength)
2. **Calculate \(t_\mathrm{MTD}\)** depending on whether track crossed:
    - **BLT**: since only 1 hit can be recorded, track time is equal to that hit time;
    - **ELT**: since 2 hits must be always recorded, defining the innermost hit as hit #1 and the outermost as hit #2:
        - propagate track from hit #2 back to hit #1, evaluating \(t_{2 \rightarrow 1} \equiv t_2 - \Delta t(2, 1)\) with \(\sigma(t_{2 \rightarrow 1}) \equiv \sigma_\mathrm{MTD} \oplus \sigma(\Delta t(\pi, p)) \) (i.e. uncertainty is inflated by TOF variation given by extreme mass hypotheses of pion and proton)
        - if \(t_1\) and \(t_{2 \rightarrow 1}\) are compatible (i.e. they pass a given \(\chi^2\) cut), their weighted average is taken as MTD time. Spatial hit corresponds to hit #1.
3. **Calculate TOF** by invoking [`computeTrackTofPidInfo()`](#computetracktofpidinfo) using the `TofCalc::kSegm` option (i.e. taking into account the variation in momentum along propagation).

#### Output
Returns `reco::Track` with same trajectory as input one (i.e. including matched MTD hit) with the evaluated time at vertex. 
Also saves the following value in the argument references:

- MTD time and relative uncertainty as defined above;
- travelled path length (as per [`trackPathLength()`](#trackpathlength));
- time of flight under \(\pi\), \(K\), \(p\) mass hypotheses with relative uncertainty (as per [`computeTrackTofPidInfo()`](#computetracktofpidinfo)).

---

### buildTrackExtra()

---

### computeTrackTofPidInfo()
``` cpp title="Prototype"

const TrackTofPidInfo computeTrackTofPidInfo(
    float magp2,
    float length,
    TrackSegments trs,
    float t_mtd,
    float t_mtderr,
    float t_vtx,
    float t_vtx_err,
    bool addPIDError = true,
    TofCalc choice = TofCalc::kCost,
    SigmaTofCalc sigma_choice = SigmaTofCalc::kCost);
```

#### Steps

Supports three TOF calculation strategies:

- `#!cpp TofCalc::kConst`: assuming constant momentum, evaluated TOF as \(\Delta t \equiv \Delta \ell / {\beta c}\);
- `#!cpp TofCalc::kSegm`: takes into account the variation of momentum due to energy loss through propagation. Uses the `TrackSegments::computeTof()`, `TrackSegments::computeSigmaTof()` methods;
- `#!cpp TofCalc::kMixd`: calculates \( \Delta t \equiv \Delta \ell / {\beta c} + \Delta t_\texttt{kSegm} \). No physical meaning, only defined for convenience of use inside `tryBTLLayers()`, `tryETLLayers()`.

The calculation is repeated for the pion, kaon and proton mass hypotheses. The associated \(\beta\) and \(\gamma^2\) is also evaluated.

The time difference with respect to the primary vertex is calculated as $$\Delta t \equiv t_\mathrm{MTD} - \Delta t_\pi - t_\mathrm{vtx}$$ corresponding to the time of the particle at the vertex. The associated uncertainty is, crucially, defined as: 

$$\sigma_{\Delta t} \equiv \sigma_\mathrm{MTD} \oplus \sigma_\mathrm{vtx} \oplus \Delta t(\pi, p)$$

where \( \Delta t(\pi, p) \equiv \Delta t_\pi - \Delta t_p \) (i.e. difference between TOF in pion and proton hypothesis). This last term is only included if `addPIDError = 1`.

??? info "Why is uncertainty evaluated this way?"
    By inflating the uncertainty on the time at vertex in this way, we are taking into account the uncertainty on the actual particle ID. This is used in the first iteration of the 4D vertex reconstruction algorithm, where different mass hypotheses cannot be resolved until a first version of vertex candidates is provided.

Lastly, a likelihood for each mass hypothesis \(\mathrm{hp}\) is evaluated as:

$$ p(\mathrm{hp}) = \frac{e^{-\chi^2_\mathrm{hp}/2}}{\sum_\mathrm{hp} e^{-\chi^2_\mathrm{hp}/2}} \hspace{0.5cm} \mathrm{ where }\; \ \chi^2_\mathrm{hp} = \frac{t_\mathrm{MTD} - \Delta t_\mathrm{hp} - t_\mathrm{vtx}}{\sigma_{\Delta t}^2}  $$

If any hypothesis performs better in terms of \(\chi^2\) with respect to the pion hypothesis, \(\Delta t\) is overriden by that hypothesis.

#### Output

Fills a `TrackTofPidInfo` struct with all the required information (see structure for list of member variables).

---

### trackPathLength()

``` cpp title="Prototype"
bool trackPathLength(const Trajectory& traj,
                     const TrajectoryStateClosestToBeamLine& tscbl,
                     const Propagator* thePropagator,
                     float& pathlength,
                     TrackSegments& trs);
```

#### Steps

!!! note
    Since it only calculates propagation *lengths*, it does not make use of any mass hypothesis. It only needs to know momentum.

Iterating over trajectory measurements (starting from outermost, first in order),

1. propagates track state to the surface of the next measurement (i.e. to the previous hit);
2. saves the calculated segment using `TrackSegments::addSegment()`, saving the propagated \(\Delta\ell\), the squared momentum magnitude at the beginning of each trajectory segment and its associated uncertainty, as obtained from `TrajectoryState::curvilinearError()` (internally calculated after track fitting through the Kalman filter algorithm);

In the end, the track segment associated to the propagation from the innermost trajectory measurement (i.e. first tracker hit) to the extrapolated beam spot is also computed.

???+ info "Used Propagator"
    Uses the `PropagatorWithMaterialForMTD` propagator, i.e. a `PropagatorWithMaterial` propagator taking into account material effects, with the following parameters:
    ``` python 
    Mass = cms.double(0.13957018),
    MaxDPhi = cms.double(1.6),
    PropagationDirection = cms.string('anyDirection'),
    ptMin = cms.double(0.1),
    useOldAnalPropLogic = cms.bool(False),
    useRungeKutta = cms.bool(False)
    ```
    Notice the usage of an analytical propagator.

#### Output
Saves sum of partial propagation lengths \(\ell\) to input argument `pathlength`.

Populates output `TrackSegments` with list of segments, each containing info on partial propagation length \(\Delta \ell\), squared momentum magnitude \(p^2\) at the beginning of each segment and uncertainty on momentum.

Returns `#!cpp bool validpropagation` checking if `#!cpp pathlength > 0` for each single propagated segment. If even a single segment fails, propagation is not considered valid.

---

### fillValueMap()

## Classes

### TrackSegments

#### Member variables
```cpp
uint32_t nSegment_ = 0;
std::vector<float> segmentPathOvc_;  // segment lengths
std::vector<float> segmentMom2_;     // squared momentum at the start of each segment
std::vector<float> segmentSigmaMom_; // uncertainty on momentum (not squared)
```

#### Methods
- `#!cpp float computeSigmaTof(float mass_inv2)`: computes \(\Delta t(m)\) as sum of partial pathlength over each segment \(\Delta t_i \equiv \texttt{segmentPathOvc_[i]} / \beta_i(m)\), with \(\beta_i(m)\) depending on the mass hypothesis `mass_inv2`;
- `#!cpp float computeSigmaTof(float mass_inv2) const`: computes \(\sigma_{\Delta t(m)}\) assuming a complete correlation between all momentum measurements, resulting in the formula 
$$
\sigma_{\Delta t(m)} = \sqrt{\sum_i \sum_j \sigma_{t_i} \sigma_{t_j}} \hspace{0.5cm} \mathrm{with} \, \sigma_{t_i} = \sigma_{p_i} \cdot \Delta \ell_i \, \frac{m^2}{p^2 E_i}
$$

using the \(\sigma_{p_i}\) as obtained in `trackPathLength()`.

??? info "\(\sigma_{\Delta t(m)}\) calculation"
    As \(\Delta t(m) = \sum_i t_i \), taking into account the pair-wise correlation between segments descending from the momentum measurement correlation:
    $$
        \sigma_{\Delta t(m)} = \sqrt{\sum_i \sum_j \left| \frac{\partial \Delta t(m)}{\partial t_i} \right| \left| \frac{\partial \Delta t(m)}{\partial t_j} \right| \, \mathrm{Cov}(t_i, t_j)}
    $$
    where \(\mathrm{Cov}(t_i, t_j) = \rho_{ij} \sigma_{t_i} \sigma_{t_j}\). We here assume all segments to be **fully correlated** (\(\rho_{ij} = 1\)) (due to the fact the Kalman filter algorithm does not return them; however, we can assume them to be strongly correlated, since the momentum variation through propagation --as well as its associated uncertainty-- is small). Partial derivatives amount to 1.
    
    Since \( t_i = \Delta\ell_i / \beta_i = \Delta\ell_i / p_i \cdot E_i = \Delta\ell_i / p_i \cdot \sqrt{p_i^2 + m^2} \), we have:
    
    $$
        \sigma_{t_i} = \left| \frac{\partial t_i}{\partial p_i} \right| \, \sigma_{p_i} = \sigma_{p_i} \cdot \Delta \ell_i \, \frac{m^2}{p^2 E_i}
    $$

    hence the formula above.


Notice both methods refer to the `kSegm` calculation strategy, and are invoked within `computeTrackTofPidInfo()`.

### TrackTofPidInfo

#### Member variables
```cpp
float tmtd;         // time of associated MTD hit
float tmtderror;    // uncertainty
float pathlength;   // travelled distance between MTD hit and beamline PCA

float betaerror;    // difference in beta between pion and proton mass hp

float dt;           // time difference between vertex and track beamline PCA
float dterror;
float dtchi2;

float dt_best;      // dt under mass hypothesis with smallest dtchi2
float dterror_best;
float dtchi2_best;

float gammasq_pi;
float beta_pi;
float dt_pi;
float sigma_dt_pi;

float gammasq_k;
float beta_k;
float dt_k;
float sigma_dt_k;

float gammasq_p;
float beta_p;
float dt_p;
float sigma_dt_p;

float prob_pi;
float prob_k;
float prob_p;
```

See [`computeTrackTofPidInfo`](#tracktofpidinfo) for how each variable is computed.

## Output

The main output quantities are here summarized again, with pointers to the method which calculates them:

<!-- make table? -->