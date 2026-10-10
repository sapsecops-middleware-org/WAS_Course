# IQ

| Real World | Mainframe World | AWS World |
|---|---|---|
| Building | Data centre | Region |
| Floor/Room | Zone / separate DC area | Availability Zone (AZ) |
| Rack (physical hardware) | The big mainframe box | Physical host (invisible to you) |
| Your slice | LPAR (Logical Partision)| EC2 Instance |

### Q1: "What is the difference between vertical and horizontal clustering and which does your bank use in production?"
```
We used a hybrid topology. Each LPAR had 2 vertical members — for resource efficiency. And we had 2 LPARs per zone, one in Zone-A and one in Zone-B — for hardware-level HA.

So a 4-member cluster looked like:

LPAR1 (Zone-A): Member A1, A2
LPAR2 (Zone-B): Member B1, B2

All critical apps — UPI, NEFT, NetBanking — followed this pattern.


Only EOD batch ran as pure vertical on a single LPAR, because it runs at night and HA wasn't a business requirement for it.
```