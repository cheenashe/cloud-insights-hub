# AWS Migration Cost Calculator

A lightweight tool to estimate your AWS spend after migrating on-premise workloads to the cloud. Input your current server specs, get a rough AWS cost projection. Built by [Sygitech](https://sygitech.com).

## Why

"What will this cost on AWS?" is the first question every migration project has to answer, and most calculators are either buried inside sales tools or too generic to be useful early on. This is a simple, transparent, open tool you can run yourself.

## What it does

- Takes basic inputs: number of servers, CPU/RAM/storage specs, current on-prem or hosting cost
- Maps to comparable EC2 instance types and estimated monthly cost
- Adds estimated costs for storage (EBS/S3), data transfer, and common managed services (RDS, load balancers)
- Outputs a simple cost comparison: current spend vs. projected AWS spend

## Usage

```bash
git clone https://github.com/sygitech/aws-migration-cost-calculator.git
cd aws-migration-cost-calculator
pip install -r requirements.txt
python calculator.py --servers servers.csv
```

Or use the web version: [calculator.sygitech.com](https://sygitech.com)

## Example input (`servers.csv`)

```
name,cpu_cores,ram_gb,storage_gb,os
web-01,4,16,100,linux
db-01,8,32,500,linux
```

## Disclaimer

This tool gives a directional estimate, not a guaranteed quote. Actual AWS costs depend on usage patterns, reserved instance/savings plan strategy, and architecture choices. For a detailed cost assessment, [talk to us](https://sygitech.com).

## Contributing

PRs to improve instance-mapping accuracy or add new AWS services to the estimate are welcome.

## About Sygitech

Sygitech specializes in AWS cloud migration and managed services, including cost optimization before and after migration. [sygitech.com](https://sygitech.com)

## License

MIT
