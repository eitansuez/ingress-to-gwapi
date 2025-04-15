# Migration

provision kgateway
 install ingress2gateway
 generate gw api artifacts from ingress api resources
 review generated artifacts
 configure kgateway for httpbin: apply generated artifacts

switch dns for httpbin to new gw ip
configure ingress for bookinfo
switch dns for bookinfo to new gw ip

decommission ingress-nginx