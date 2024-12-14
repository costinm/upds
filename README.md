# uPDS

Experiments with BSky PDS and concepts - adapted to use in non-social contexts. 

I am exploring the use of a repo of signed objects, similar to what is used for ATproto posts,
but used for configs and telemetry instead of social messages. 

This is an old idea - I tried it first with Webpush-encrypted messages, as a way to scale push messaging
using non-trusted middle boxes (that was for GCM/FCM/chrome webpush - with many B devices), also
played with it with signed XDS messages in Istio, but it was too complex and hard to move well-established
infra. A subset of ATproto could be a cleaner and more viable path. 

Why ? I think it is the missing piece for improved scalability and security, not only for K8S or Istio
but most enterprise servers. The model of a 'control plane' that is fully trusted works well, but
it is better to have all configs signed by original author - without the need to trust the control plane.
The difficult part is the modifications of configs - but keeping the history of the config can 
help not only tracking the change but also recovering to a previous config version. Similar for telemetry -
there is a lot of implicit trust in many middle boxes, at least for logs. 

ATproto is a set of protocols: 'sync' appears to be the required one for publishing messages into the 
network and distribution at scale - most of the others are required to use the BSky app UI which 
is only used for the social case. It is still nice to verify what is strictly required for ATproto. At 
some point it may be nice to also use the protocol for messages (not necessarily social or only 
between humans), like logs/alerts/etc.

I suspect the Lexicon, xrpc, CBOR and other elements of the protocol can be replaced/augmented with 
gRPC and protobuf or with OpenAPI and json, doesn't appear to be essential to the design and 
it is good to have an alternative ( still nice to keep them as a possible transport, for validation).

# Identity

K8S and mesh are already based on keypairs - only need to support EC256 and instead of PLC can use
regular certificates. I think it's useful to also support the PLC and some interop - so tools and
cool apps for ATproto can be adapted - but it doesn't seem required to support all options.
A PLC variat, as a way to track and distribute workload and service public keys and 
handle rotations - without the requirement of a mesh-wide CA - is interesting to explore, but 
not a priority - mesh CAs are stable enough.

Authentication will use mTLS and JWTs - in addition or instead of the custom version of OAuth or OIDC 
that ATproto uses (with normal url in JWTs, not dids). Similarly, normal URLs for services and 
identifiers, I'm primarily interested in signed objects, not to replace well-established things.

# Centralization

K8S and Istio can run fully decoupled from Internet. A couple of RaspberryPi - and you have 
working DNS, secure network, CA, control plane and so on, no internet needed besides downloading
the images. Running ATproto in such environment can be a proof that it is indeed a decentralized
protocol ( but not as a 'social protocol' !). Assuming that a network protocol will only be used
for a specific use case - small social messages or 'hypertext', or designing a protocol that 
can be only be used for one use case is not ideal.

K8S and Istio in a cluster are quite centralized - the APIserver, Istiod, CA are critical
pieces, if any is compromised or down it will impact the entire cluster. This is handled by 
having replicas - but the risk of a node compromise impacting a critical component remains
(and is handled in various ways, separate node pools, etc). In multi-cluster cases - things
are even more interesting on the trust model and risks.

A 'signed config' model - where each operation is signed and then replicated on each node - no
longer has a SPOF or require trusting any specific node. If the config changes are 
infrequent - including deployments - the system can keep working, and the signing can
happen in a separate environment. 

Endpoints and Workload identities are more complicated but if a Pod is assigned to a node, with
a signed config verifying the assignment, the node can sign and handle the Pod identity and 
endpoint changes. Each Node in this model will have a signing key and their own identity, with
a per-node PDS server.

The role of K8S and Istiod (plus a specialized controller) will be to distribute signed config
to nodes, with PDS repo sync allowing node-to-node sync if the control plane is out. 
Of course it won't work for everything - but I think for most Istio configs and 
core K8S it would be good enough. 


# Primary or replica PDS servers

ATproto is mainly based on each user having a single PDS server active (which may be multi-tenant and 
handle many users). Migration means the user moves to a different PDS server. The PDS servers hold the 
signing keys and as such need to be trusted.

If instead the signing and keys are held on a separate server (control plane or the 'source of truth' - 
ex. a git repo) - the signed data can be on multiple PDS servers with no special trust in any. 

A 'primary' server is still needed if using BSky app for social apps - but not for the use of ATproto 
for distributed signed objects. Even for the social UI, the PDS used by the UI can be synchronized 
with multiple replica servers and any of the replicas can be used for reading. Not clear if 
multiple writers are possible, but I suspect it can work too, at least for separate feeds.

# Plan

Start with a ATproto golang SDK and only expose the minimal sync, with an alternative gRPC or OpenAPI 
interface for write/append/etc - as well as sync using typical 2-way streaming and watch. 

The PDS server will hold NO private key - signing of posts will be separate (by the resource owner or 
the control plane that does the writes). A specialized, more trusted instance could read directly from K8S 
or XDS and generate the repo feed - but local PDS servers will just cache and distribute the feeds.

