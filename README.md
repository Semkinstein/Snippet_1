import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Client } from 'amqplib';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject } from '@nestjs/common';
import { Cache } from 'cache-manager';

interface BorrowerInfo {
  id: string;
  score: number;
  history: any[];
}

@Injectable()
export class BorrowerService {
  constructor(
    @InjectRepository(BorrowerEntity)
    private borrowerRepository: Repository<BorrowerEntity>,
    @Inject('RABBITMQ_CLIENT') private rabbitmqClient: Client,
    @Inject(CACHE_MANAGER) private cacheManager: Cache
  ) {}

  async getBorrowerData(borrowerId: string): Promise<BorrowerInfo> {
    const cachedData = await this.cacheManager.get<BorrowerInfo>(`borrower_${borrowerId}`);
    if (cachedData) {
      return cachedData;
    }


    const borrower = await this.borrowerRepository.findOne({
      where: { id: borrowerId },
      relations: ['loans', 'creditHistory']
    });

    if (!borrower) {
      throw new Error('Borrower not found');
    }


    const result = {
      id: borrower.id,
      score: borrower.creditScore,
      history: borrower.creditHistory
    };

    await this.cacheManager.set(`borrower_${borrowerId}`, result, 300000);

    this.rabbitmqClient.sendToQueue(
      'data_refresh',
      Buffer.from(JSON.stringify({ borrowerId, timestamp: Date.now() }))
    );

    return result;
  }
}
