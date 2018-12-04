# BAREB R package

This package provides a tool to implement the BAREB algorithm: a Bayesian repulsive biclustering model for periodontal data. 

## Installation


You can manually obtain BayRepulsive.tar.gz and install it directly:

```
R CMD install BAREB.tar.gz
```

Note that the package use R package "Rcpp" and "RcppArmadillo"

If this is the first time you use Rcpp, you might find the following link helpful.

https://thecoatlessprofessor.com/programming/r-compiler-tools-for-rcpp-on-macos/

## Usage

```R
rm(list=ls())
library(BAREB)
data("simobs")
data("simtruth")

set.seed(1)

n<-80
m<-168
T0<-28
q<-3
p<-3
S<-3
theta1 <- theta2 <- 5
tau <- 100000
D<-matrix(0, nrow = T0, ncol = m)
for(i in 1:T0){
indi<-1:6
indi<-indi+6*(i-1)
D[i,indi]<-rep(1/6,6)
}
nu_gamma<-0.05
nu_beta <- 0.05
data("initial")

Niter<-5000
record <- NULL
record$E<-matrix(NA,nrow = Niter, ncol = n)
record$R<-array(NA, dim = c(Niter, S, m))
record$Gamma <-array(NA,dim = c(Niter, 10, q, S))
record$Beta <- array(NA,dim = c(Niter, S, p))
record$K <- matrix(NA,nrow = Niter, ncol = S)
record$sigma_square <-rep(NA,Niter)
record$theta1<-rep(0,Niter)
record$theta2<-rep(0,Niter)
record$c<-matrix(0,nrow =  Niter,ncol = T0)
record$mu<-array(NA,dim = c(Niter,n,m))
record$w_beta <- array(NA, dim = c(Niter, S))
record$w <- array(NA, dim = c(Niter, S, 10))
set.seed(1)
E<-sample.int(S,n,T)
Beta <- matrix(NA,nrow = S, ncol = p)
Beta[1,] <- Beta[2,] <- Beta[3,]  <- Beta0
Beta<- Beta + matrix(rnorm(S*p, 0, 1) , nrow = S, ncol = p)
Gamma <- array(NA,dim = c(10,q,S))
Gamma[1,,1] <- Gamma[2,,1] <- Gamma[3,,1] <- Gamma0
Gamma[,,2] <- Gamma[,,3]   <- Gamma[,,1]
Gamma <- Gamma + array(rnorm(10*q*S, 0, 5), dim = c(10, q, S))
K <- rep(3,S)
R <- matrix(NA,nrow = S, ncol = m)
R[1,]<- R[2,]<- R[3,]  <- sample.int(3,m,T)
mu<-updatemu(R,Z,X,Gamma,K,Beta,E,m,n,p,q)
mu_star<-updatemustar(mu,rep(0.01,T0),n,T0,D)
z_star<-updateZstar(mu_star,delta,n,T0)
sigma_square <- 10
C<-array(NA,dim = c(10,10,S))
w<-matrix(NA,nrow = S, ncol = 10)
w_beta<-rep(1/S,S)
for(i in 1:S){
C[1:K[i],1:K[i],i]<-updateC(Gamma[1:K[i],,i],theta2,tau)
w[i, 1:K[i]]<-rep(1/K[i],K[i])
}
c<-0.01
start <- Sys.time()
for(iter in 1:Niter){
c<-updatec(z_star, mu,D, 100, n, T0)
mu_star<-updatemustar(mu,rep(c,T0),n,T0,D)
z_star<-updateZstar(mu_star,delta,n, T0)
w <- update_w(K, w, R, S)
R <- updateR(w, Gamma, Beta,
Y, Z, delta, mu, mu_star, c, S,
sigma_square, K, E, X,
m, n, q, p, T0)
for(i in unique(E)){
ind<-sort(unique(R[i,]))
KK<-length(ind)
Gamma_temp<-Gamma[ind,,i]
Gamma[,,i]<-NA
Gamma[1:KK,,i]<-Gamma_temp
w_temp<-w[i,ind]
w_temp<-w_temp/sum(w_temp)
w[i,]<-NA
w[i,1:KK]<-w_temp
for(k in 1:KK){
R[i,which(R[i,]==ind[k])]<-k
}
K[i]<-KK
}
mu<-updatemu(R,Z,X,Gamma,K,Beta,E,m,n,p,q)
mu_star<-updatemustar(mu,rep(c,T0),n,T0,D)
step <- array(rnorm(max(K) * S *q, 0, nu_gamma),dim=c( max(K), q,S))
run<- array(runif(max(K) * S *q, 0, 1),dim=c( max(K), q,S))
A<-updateGamma(X,Y, Z, delta, Beta, Gamma, E, R, S, K , mu, mu_star, sigma_square, rep(c,T0),
step,  run,  n,  m, T0,  p,  q,  D,theta2,  tau)
Gamma<-A$Gamma
mu<-A$mu
mu_star<-A$mustar
for(i in 1:S){
if(K[i]==1){
Gammai = t(as.matrix(Gamma[1:K[i],,i]))
C[1:K[i],1:K[i],i]<-updateC(Gammai,theta2,tau)
}
else{
C[1:K[i],1:K[i],i]<-updateC(Gamma[1:K[i],,i],theta2,tau)
}
}
A<-update_RJ(w, K, Gamma,Beta,
Z, X, R, mu, mu_star, Y, delta, c,sigma_square, C,
S, theta2)


K<-A$K
w<-A$w
Gamma<-A$Gamma
R<-A$R
C<-A$C
mu<-updatemu(R,Z,X,Gamma,K,Beta,E,m,n,p,q)
mu_star<-updatemustar(mu,rep(c,T0),n,T0,D)
C_beta<-updateC(Beta,theta1,tau)
step<-matrix(rnorm(S*p,0,nu_beta),nrow = S)
runif<-matrix(runif(S*p,0,1),nrow = S)
A<- updateBeta( X,Y, Z, delta,
Beta, Gamma, E,R,S,K,mu_star, mu,sigma_square,rep(c,T0), C_beta, step,runif,
n, m, T0,  p,  q,  D,
theta1, tau)
Beta<-A$Beta
mu<-A$mu
mu_star<-A$mustar
record$Beta[iter,,]<-Beta
A<-updateE( Beta,Gamma, w_beta, X,Y,Z,delta,E, R, S,K, mu, mu_star,sigma_square,rep(c,T0),
n,  m,  T0,  p,  q,  D)
E<-A$E
mu<-A$mu
mu_star<-A$mustar
K<-A$Ds
record$K[iter,]<-K
record$R[iter,,]<-R
record$Gamma[iter,,,]<-Gamma
record$E[iter,]<-E
record$mu[iter,,] <- mu
w_beta<- update_w_beta(S, w_beta, E)
record$w[iter,,] <- w
record$w_beta[iter,] <- w_beta
theta1<-update.theta.beta(theta1,tau,Beta)
theta2<-update.theta.gamma(theta2,tau,Gamma,S,K)
sigma_square <- update_sigma_squre(Y,mu)
record$sigma_square[iter] <- sigma_square
}


```

Full documentation and usage information is available in the manual:

* [BayRepulsive.pdf](https://github.com/bruce1995/BayRepulsive/blob/master/BayRepulsive.pdf)


## License

This library is made available under the Johns Hopkins University License. This is an open-source project of Yanxun's lab, with collaborator Dr. Dipankar Bandyopadhyay.
