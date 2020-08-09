# rexec_cvc4
Remote Exec CVC4



| Branch match? | Commit match? | Local Modified? | Remote Modified? | Was -s sync used? | Ideal:        | Current:      | Desired:                | Notes                                          |
|---------------|---------------|-----------------|------------------|-------------------|---------------|---------------|-------------------------|------------------------------------------------|
| Yes           | Yes           | Yes             | N/A              | Yes               | (none)        | (none)        | (none)                  |                                                |
| Yes           | Yes           | Yes             | N/A              | No                | -s src        | (none)        | warning: use -s         | OK if local has no changes since last -s       |
| Yes           | No            | Yes             | N/A              | N/A               | -r ; -s src   | error: use -r | force -r, error: use -s |                                                |
| No            | No            | Yes             | N/A              | N/A               | -b X ; -s src | error: use -b | force -b, error: use -s |                                                |
| Yes           | Yes           | No              | Yes              | N/A               | -r            | (none)        | force -r                | Only happens if local reverts changes after -s |
| Yes           | Yes           | No              | No               | N/A               | (none)        | (none)        | (none)                  |                                                |
| Yes           | No            | No              | N/A              | N/A               | -r            | error: use -r | force -r                |                                                |
| No            | No            | No              | N/A              | N/A               | -b X          | error: use -b | force -b                |                                                |
